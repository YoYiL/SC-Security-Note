> **
>
> **注意 / NOTE**: 这是一个用来记录我在智能合约安全中学习和 debug 经验的笔记，仅供参考。  
> This is a note used to record my learning and debugging experience in smart contract security, for reference only.
>
> 📋 **目录导航 / Table of Contents**: 点击右上角的目录图标 (📋) 查看完整目录  
> Click the table of contents icon (📋) in the upper right corner to view the complete directory
>
> Author: YoYiL


- [Dangerous Functions](#dangerous-functions)
  - [selfdestruct()](#selfdestruct)
    - [The Unique Characteristic of Selfdestruct](#the-unique-characteristic-of-selfdestruct)
    - [重大变化：EIP-6780 的影响](#重大变化eip-6780-的影响)
    - [EIP-6780 的核心规则](#eip-6780-的核心规则)
      - [情况1：同一交易中创建和销毁（完全有效）](#情况1同一交易中创建和销毁完全有效)
      - [情况2：不同交易中销毁（功能受限）](#情况2不同交易中销毁功能受限)
    - [当前状态和弃用](#当前状态和弃用)
    - [当前可能的攻击向量](#当前可能的攻击向量)
      - [1. 强制发送以太币攻击](#1-强制发送以太币攻击)
      - [2. 同交易创建-销毁攻击](#2-同交易创建-销毁攻击)
      - [3. 状态不一致攻击](#3-状态不一致攻击)
    - [防护措施](#防护措施)
      - [1. 避免依赖合约销毁](#1-避免依赖合约销毁)
      - [2. 余额检查保护](#2-余额检查保护)
      - [3. 状态管理](#3-状态管理)
      - [4. 访问控制强化](#4-访问控制强化)
- [Attack Vectors](#attack-vectors)
- [Security Review](#security-review)
  - [What is a Smart Contract Audit?](#what-is-a-smart-contract-audit)
    - [The Three Phases of a Security Review](#the-three-phases-of-a-security-review)
    - [Reach Out for a Review](#reach-out-for-a-review)
    - [Initial Report](#initial-report)
    - [Mitigation Phase](#mitigation-phase)
    - [Final Report](#final-report)
    - [Ensuring a Successful Audit](#ensuring-a-successful-audit)
    - [Post Audit](#post-audit)
    - [What an audit *isn't*](#what-an-audit-isnt)
  - [Embedding Security Audits in Development Lifecycle](#embedding-security-audits-in-development-lifecycle)
  - [Rekt Test](#rekt-test)
    - [Audit Readiness](#audit-readiness)
    - [The Rekt Test](#the-rekt-test)
    - [Nascent Audit Readiness Checklist](#nascent-audit-readiness-checklist)
  - [Tools for Security Reviews](#tools-for-security-reviews)
    - [Static Analysis: Debugging Without Execution](#static-analysis-debugging-without-execution)
    - [Fuzz Testing: Randomness Meets Tests](#fuzz-testing-randomness-meets-tests)
    - [Formal Verification: Mathematical Proofs](#formal-verification-mathematical-proofs)
    - [AI Tools: Not Quite There Yet](#ai-tools-not-quite-there-yet)
  - [What If Your Security Audit Fails?](#what-if-your-security-audit-fails)
    - [Redefining the Role of Auditors](#redefining-the-role-of-auditors)
    - [Who Owns the Blame?](#who-owns-the-blame)
    - [The Auditor’s Role in the Wake of a Breach](#the-auditors-role-in-the-wake-of-a-breach)
- [PasswordStore](#passwordstore)
  - [Scoping](#scoping)
    - [Scoping：Etherscan](#scopingetherscan)
    - [Scoping: Audit Details](#scoping-audit-details)
        - [Preparing for the Audit: Onboarding Questions](#preparing-for-the-audit-onboarding-questions)
        - [Scope](#scope)
    - [Scoping: CLOC(*Count Lines of Code*)](#scoping-cloccount-lines-of-code)
      - [The Importance of Knowing Your Codebase Size](#the-importance-of-knowing-your-codebase-size)
  - [The Tincho Auditing Method](#the-tincho-auditing-method)
    - [First Step](#first-step)
    - [Tools and Frameworks](#tools-and-frameworks)
    - [Audit, Review, Audit, Repeat](#audit-review-audit-repeat)
    - [Communication](#communication)
    - [Wrapping it Up](#wrapping-it-up)
    - [The Audit Report and Follow Up](#the-audit-report-and-follow-up)
    - [Aftermath of a Missed Vulnerability](#aftermath-of-a-missed-vulnerability)
  - [Reconnaissance](#reconnaissance)
    - [Recon: Context](#recon-context)
    - [Recon: Understanding the Code](#recon-understanding-the-code)
  - [Exploit](#exploit)
    - [Exploit: Accsess Control](#exploit-accsess-control)
    - [Exploit: Public Data](#exploit-public-data)
- [Puppy Raffle](#puppy-raffle)
- [TSwap](#tswap)
- [Thunder Loan](#thunder-loan)
- [Boss Bridge](#boss-bridge)
- [MEV \& Governance](#mev--governance)




# Dangerous Functions

## selfdestruct()

### The Unique Characteristic of Selfdestruct

Why `selfdestruct` stands out lies in its exceptional behavior once a contract gets destroyed. Any Ethereum (or ETH) residing within the deleted contract gets automatically ‘pushed’ or ‘forced’ into any address that you specify.

Under normal circumstances a contract that doesn't contain a receive or fallback function (or some other payable function capable of receiving funds) cannot have ETH sent to it.

Only through the use of `selfdestruct` can you be permitted to push any Ethereum into such a contract.

So if ever you’re hunting for an exploit, or you have identified an attack where you need to force ETH into a contract, `selfdestruct` will be your instrument of choice.

### 重大变化：EIP-6780 的影响

**从 Cancun 硬分叉开始，底层操作码不再删除代码和数据** 。EIP-6780 显著削减了 SELFDESTRUCT 操作码的功能 。

**Ethereum Dencun 升级引入的 EIP-6780 更新了 selfdestruct 操作码，停用了合约的销毁功能** 。这意味着：

- 合约代码不再被实际删除
- 合约存储不再被清除
- 只有余额转移功能保留

### EIP-6780 的核心规则

**EIP-6780 建议的改变是 SELFDESTRUCT 只能在创建合约的同一交易中被调用** 。这意味着：

#### 情况1：同一交易中创建和销毁（完全有效）

**当 SELFDESTRUCT 在部署合约的同一交易中执行时，这个特定的 EIP 保留了操作码的完整功能** 。

**具体例子：**

```
// 在一个交易中：
// 1. 部署合约A
// 2. 立即调用合约A的selfdestruct()
// 结果：合约完全被销毁，代码和状态都被移除
```

#### 情况2：不同交易中销毁（功能受限）

**Ethereum Dencun 升级引入了 EIP-6780，更新了 selfdestruct 操作码，停用了合约的销毁功能** 。

**具体例子：**

```
// 交易1：部署合约A
// 交易2：调用合约A的selfdestruct()
// 结果：只转移余额，但合约代码和状态保留（变成"僵尸合约"）
```

### 当前状态和弃用

**SELFDESTRUCT 已经被弃用了一段时间** 。在 Solidity 0.8.24 版本中，编译器会对使用 `selfdestruct` 发出弃用警告 。

**"selfdestruct" 已被弃用。注意，从 Cancun 硬分叉开始，底层操作码不再删除代码和数据** 。

### 当前可能的攻击向量

#### 1. 强制发送以太币攻击

**操作码仍然转移以太币** ，这意味着攻击者仍然可以：

- 强制向目标合约发送以太币
- 破坏依赖精确余额计算的合约逻辑
- 绕过某些访问控制机制

#### 2. 同交易创建-销毁攻击

由于 **EIP 在合约创建的同一交易中执行 SELFDESTRUCT 时保留了操作码功能** ，攻击者可以：

- 在同一交易中部署恶意合约并立即销毁
- 利用这种模式进行复杂的攻击
- 规避某些检测机制

#### 3. 状态不一致攻击

**以太坊已在 Cancun 升级中纳入了 EIP-6780，修改了 SELFDESTRUCT 操作码的行为** ，但由于余额转移功能保留，可能导致：

- 合约状态与实际余额不匹配
- 依赖合约"死亡"状态的逻辑被破坏

### 防护措施

#### 1. 避免依赖合约销毁

- 不要假设合约会被完全移除
- 设计时考虑合约可能"僵尸化"

#### 2. 余额检查保护

- 实现robust的余额验证逻辑
- 不依赖精确的余额计算

#### 3. 状态管理

- 使用明确的状态标记而非依赖合约存在性
- 实现暂停/停用机制

#### 4. 访问控制强化

- 不依赖合约不存在作为安全保证
- 实现多层访问控制



# Attack Vectors





# Security Review



## What is a Smart Contract Audit?

Let's start off by stating that the term "smart contract audit" is a bit of a misnomer. As a more appropriate alternative, I am a stout advocate of "security review." I even have a T-shirt to prove my allegiance!

You might be wondering why this change of terms is required. Well, it’s because the term 'audit' might wrongly insinuate some kind of guarantee or even encompass legal implications. A security review, being free of these misconceptions, exudes the essence of what we are actually doing: looking for as many bugs as possible to ensure maximum code security.

> Note: Despite this, many protocols still insist on requesting a "smart contract audit," so it's eminent to know that the terms are interchangeable. When you hear "security review", think "smart contract audit" and vice versa. Protocols are often unaware of these nuances, but you, as a trained security researcher, know better!

### The Three Phases of a Security Review

- [High Level Overview](https://www.youtube.com/watch?v=aOqhQvWhUG0)
- People say "audit" -> security review
- There is no silver bullet to auditing, and they have limitations
- 3 phases of a security review
  - Initial Review
    - 1. Scoping
    - 1. Reconnaissance（侦察）
    - 1. Vulnerability identification
    - 1. Reporting
  - Protocol fixes
    - 1. Fixes issues
    - 1. Retests and adds tests
  - Mitigation Review
    - 1. Reconnaissance
    - 1. Vulnerability identification
    - 1. Reporting

Often a single audit won't be enough, protocols are really entering into a security journey which may include:

- Formal Verification
- Competitive Audits
- Mitigation Reviews
- Bug Bounty Programs

### Reach Out for a Review

The review process begins when a protocol reaches out, be it before or after their code is complete. After they make contact, it's important to determine the cost of a review based on things like:

- Code Complexity/nSLOC
- Scope
- Duration
- Timeline

Lines of Code: Duration

- 100 : 2.5days
- 500 : 1 Week
- 1000 : 1-2 Weeks
- 2500 : 2-3 Weeks
- 5000 : 3-5 Weeks
- 5000+: 5+ weeks

Take this with a lot of salt though, as these timelines vary largely based on circumstance.

With the submission of a `commit hash` and `down payment` by the protocol and start date can be set!

> Note: The `commit hash` is the unique ID of the codebase an auditor will be working with.

### Initial Report

Once the review period is over, the auditors compile an initial report. This report includes all findings, categorized according to severity

- High
- Medium
- Low
- Information/Non-critical
- Gas Efficiencies

High, medium and low findings have their severity determined by the impact and likelihood of an exploit.

Informational/Non-Critical and Gas are findings focused on improving the efficiency of your code, code structure and best practices. These aren't vulnerabilities, but ways to improve your code.

### Mitigation Phase

The protocol's team then has a fixed period to address the vulnerabilities found in the initial audit report. More often than not, they can simply implement the recommendations provided by the auditors.

### Final Report

Upon completion of the mitigation phase, the audit team compiles a final audit report focusing exclusively on the fixes made to address the initial report's issues. Hopefully, this cements a strong relationship between the protocol and the audit team, fostering future collaborations to keep Web3 secure.

### Ensuring a Successful Audit

For an audit to be as successful as possible, you should ensure that there's:

- Good documentation
- A solid test suite
- Clear and readable code
- Modern best practices are followed
- Clear communication channels
- An initial video walkthrough of the code

By considering auditors as an extension of your team, maintaining an open channel of communication, and providing them with the necessary documentation and context, you ensure the audit process is smoother and more accurate, providing auditors valuable context of the codebase.

### Post Audit

Lastly, remember that a smart contract audit is an integral part of a security journey rather than an endpoint. Even after an audit, any subsequent code changes need to be reviewed as the new code is unaudited, regardless of the size of the change.

> Remember: One audit might not be enough. Getting more eyes on your code is only going to increase the chances of catching vulnerabilities before it's too late

### What an audit *isn't*

Going through a security review does not mean that your code is bug free. Security is a continuous process tha tis always evolving.

> "There is no silver bullet in smart contract auditing. But understanding the process, methods, and importance of regular security reviews can significantly enhance your protocol's robustness."

![the-audit1](SC-Security-note.assets/the-audit1.png)

## Embedding Security Audits in Development Lifecycle

The process of developing a smart contract follows a lifecycle too. According to the [OWASP](https://www.owasp.org/index.php/Main_Page) (The Open Web Application Security Project) guide, security isn't just a one-off step but a part of your ongoing smart contract journey. It is about fostering the mindset that security is continuous. The smart contract developer lifecycle entails the following stages:

1. **Plan and Design**
2. **Develop and Test**
3. **Get an Audit**
4. **Deploy**
5. **Monitor and Maintain**

OWASP strongly emphasizes that embedding security considerations into all stages of your Development Lifecycle is what it takes to build a secure decentralized application, not just conducting a one time smart contract “check.” Before deploying your contract, think hard about the security measures in place and ensure to maintain and monitor your code post-deployment.

While a smart contract security audit is an absolute necessity, also ensure to plan for any contingencies post-deployment. The key takeaway here is this: Smart contract security is a crucial part of the smart contract development lifecycle and should be treated with as much care as the development of the smart contract itself.

## Rekt Test

*'Rekt' 在这里是拼写变体，源自英语俚语 'wrecked'（毁灭）*

### Audit Readiness

The concept that once you've had an audit done, you're ready to ship - is wrong. There are two tests that I tell everyone to look at prior to getting a security review one is the [**nacentxyz simple-security-toolkit**](https://github.com/nascentxyz/simple-security-toolkit) and the other is [**The Rekt Test**](https://blog.trailofbits.com/2023/08/14/can-you-pass-the-rekt-test/), by Trail of Bits.

### The Rekt Test

The Rekt Test is highly important as it poses a set of questions to gauge your protocol's preparedness for an audit. This tool forces you to think about security measures from a more proactive angle. Should your protocols fail to answer these questions, the chances are that they're not audit-ready.

The questions touch on several aspects like documentation, security roles, security tools, and protective measures, among others. Here's a curated list:

1. **Do you have all actors roles and privileges documented?**
2. **Do you keep documentation of external services contracts and oracles?**
3. **Do you have a written and tested incident response plan?**
4. **Do you document the best ways to attack your system?**
5. **Do you perform identity verification and background checks on all employees?**
6. **Do you have a team member with security defined in the role?**
7. **Do you require hardware security keys for production systems?**
8. **Does your key management system require multiple humans and physical steps?**
9. **Do you define key invariants for your system and test them on every commit?**
10. **Do you use the best automated tools to discover security issues in your code?**
11. **Do you undergo external audits and maintain a vulnerability disclosure or bug bounty program?**
12. **Have you considered and mitigated avenues for abusing users of your system?**

As developers, you must be able to answer all these queries before you proceed with an audit. If you're dealing with a protocol that fails to answer these questions, it's best to tell them the protocol isn't ready to ship, or arguably audit, until they can.

> "Delegate responsibility to someone on your team for security - Give your project a sense of ownership and a point person to handle any security breaches."

### Nascent Audit Readiness Checklist

[**This**](https://github.com/nascentxyz/simple-security-toolkit) checklist is another effective method to assess if you're ready for an audit. Though it offers different perspectives, it's another tool that helps you determine if your protocols are prepared for audits.

## Tools for Security Reviews

### Static Analysis: Debugging Without Execution

Static analysis represents the next level of defense. This method automatically checks for issues without executing your code, hence the debugging process remains `static`. Slither, 4nalyzer, Mythril, and Aderyn are some prominent tools in the static analysis category.

Throughout this course, we'll work heavily with Slither and Aderyn, you'll become experts at these static analysis options.

### Fuzz Testing: Randomness Meets Tests

Next we have Fuzz testing, which really comes in two flavours, `fuzz testing` and `stateful fuzz testing`.

![tools2](SC-Security-note.assets/tools2.png)

A few other types of testing we *won't* be covering are `differential test` and `chaos tests`, but in an effort to further you security journey, you always want to be looking for new looks and expanding your knowledge, so you may want to check them out.

### Formal Verification: Mathematical Proofs

Formal verification is a broad term for deploying formal methods to affirm the correctness of hardware or software. Often, these methods involve converting the codebase into mathematical expressions and deploying mathematical proofs to authenticate that the code does or doesn't do something specific.

A popular formal verification approach is symbolic execution. This method converts your Solidity function into math or a set of boolean expressions. Manticore, Certora, Z3 stand tall in this domain.

We will delve deeper into formal verification in later sections.

### AI Tools: Not Quite There Yet

Lastly but importantly, AI tools offer another dimension to imagine code auditing functionalities. However, despite their potential, they have some distance to cover before they provide substantial value for securing a codebase. At present, using AI tools could serve as a sanity check or aid in looking for something quickly, but if a project suggests it has been audited by an AI tool like `ChatGPT`, it is best to be skeptical and question if the project takes security seriously.

There's a great GitHub repo by ZhangZhuoSJTU that illustrates examples of bugs that are detectable by machines and those that aren't. Check it out [**here**](https://github.com/ZhangZhuoSJTU/Web3Bugs).

## What If Your Security Audit Fails?

### Redefining the Role of Auditors

In the eyes of many, the fundamental purpose of a security audit is to identify and rectify the most critical vulnerabilities in a system. However, Tincho encourages us to look beyond this simplistic view.

> Auditors should provide value, regardless of whether or not they spot critical issues.

In other words, an auditor's value doesn't solely rest upon their ability to find vulnerabilities. Instead, their advice should strengthen the overall security protocol and offer pragmatic solutions for future scenarios.

Of course, it goes without saying that the fewer critical vulnerabilities that are overlooked, the better - the safer Ethereum will be. It's naive however to believe that an auditor is solely responsible for when things go wrong.

### Who Owns the Blame?

The notion of finding a scapegoat when a system is exploited is a regressive one.

> A whole chain of events leads to the successful exploitation of a vulnerability.

Attributing the failure of a system to an auditor's incompetency is simplistic and misguided. If a vulnerability was missed, it means it slipped past numerous stages of checks and balances, of which an audit is just one. When a flaw goes unnoticed for as long as four months, there are perhaps lapses in system monitoring and in many other security parameters.

### The Auditor’s Role in the Wake of a Breach

So, what should an auditor do if a protocol they've reviewed ends up compromised? The answer is that a responsible security partner should not abandon their client in the midst of a crisis.

As an auditor, you may be able to help mitigate the damage, restrict the scope of the attack, and possibly identify the hackers. A quality auditor must be there, lending their expertise, during the inevitable chaos that ensues after a breach.

> "If you are to be the trusted security partner of your clients, probably, when they are hacked, you want to be there. You want to be there supporting them." - Tincho





# PasswordStore

## Scoping

### Scoping：Etherscan

As security researchers, you're looking for more than bugs. You're looking for code maturity. If all you have is a codebase on etherscan, if there's no test suite, if there's no deployment suite you should be asking: `how mature is this code?`

> **Remember: Secure protocols not only safeguard the code but also our reputation as researchers. They will likely blame us for a security breach if we've audited a compromised codebase.**

If all they provide is an etherscan link, can you assure the protocol's safety? In these cases, the answer is a resounding **NO**.

![rekt1](SC-Security-note.assets/rekt1.png)

If all they've provided you is an Etherscan link - the answer is poorly.

> **If you're offered monetary reward to audit an Etherscan-only codebase, that's a red flag. Say NO. Doing otherwise contradicts our mission to promote secure protocols.**

Do not take clients who have not shown the same commitment to security in their codebase as you would. If you work with clients like those described above, it should be to educate them on how to write good tests and how to prepare their code for a review.



### Scoping: Audit Details

##### Preparing for the Audit: Onboarding Questions

For your convenience, we've compiled a reference of [**Minimal Onboarding Questions**](https://github.com/Cyfrin/security-and-auditing-full-course-s23/blob/main/minimal-onboarding-questions.md). This document will help you extract the minimum information necessary for a successful audit or security review.

We've also included a more [**Extensive Onboarding Questions**](https://github.com/Cyfrin/security-and-auditing-full-course-s23/blob/main/extensive-onboarding-questions.md) document which is more derivative of what we at Cyfrin use for private audits - we'll go over this in more detail later.

Let's go through these questions and understand why each one is important in preparing for our security review.

1. **About the Project:** Knowledge about the project and its business logic is crucial. You need to be aware of what the project is intended to do so as to spot areas where code implementation does not align with the project's purpose. Remember 80% of vulnerabilities are a product of business logic implementation!
2. **Stats:** Information about the size of the codebase, how many lines of code are in scope, and its complexity are incredibly vital. This data will help to estimate the timeline and workload for the audit.
3. **Setup:** We need to ask the protocol how to build and test the project, which frameworks they've used etc.
4. **Review Scope:** Know the **exact commit hash** that the client plans to deploy and the **specific elements of the codebase** it covers. You do not want to spend time auditing code that the client has already modified or doesn't plan to use. The protocol should include the appropriate GitHub URL and explicitly detail which contracts are in scope.
5. **Compatibilities:** Information about the solidity version the client is using, the chains they plan on working with, and the tokens they will be integrating is important, we'll go into why later.
6. **Roles:** This entails understanding the different roles and powers within the system and detailing what the different actors should and shouldn't be able to do.
7. **Known Issues:** Understanding existing vulnerabilities and bugs which are already being considered/fixed. This will allow you to focus on the hidden issues.

##### Scope

For this particular example, the client has provided scope:

```
./src/

└── PasswordStore.sol
```

In this case, a single contract - depending on the maturity of the protocol, you may want to request to include their deployment process, or to provide feedback on their tests - but this is largely a private audit consideration. ==**In competitive audits, the outlined scope is the only code that will be valid.**==

### Scoping: CLOC(*Count Lines of Code*)

**Stats** *Use something like solidity metrics or cloc to get these.*

- nSLOC: XX
- Complexity Score: XX
- Security Review Timeline: Date -> Date

One of the components of the `Stats` section is `nSLOC` or `number of source lines of code`. A very simple tool exists to help us derive this count.

[**CLOC**](https://github.com/AlDanial/cloc) - cloc counts blank lines, comment lines, and physical lines of source code in many programming languages. It's compatible with Solidity, Python, Rust and many more.

![image-20250913172004494](SC-Security-note.assets/image-20250913172004494.png)

#### The Importance of Knowing Your Codebase Size

As you perform more audits and delve further into security research, you'll start to gauge the pace at which you can audit a code base. Understanding that pace enables you to estimate more accurately the time required for future coding or auditing tasks based on the size of the code base.

This is incredibly useful, as with time, you can use your past audit experience and tell the protocol you're working with how long it will take to audit their codebase. Notably, this pace tends to speed up as you do more security reviews. Nevertheless, it's a good starting point.

> *"When auditing 1000 lines of code for the first time, you now have an estimated timeline for subsequent audits or security reviews of 1000 lines codebases."*

Often, competitive audits might have a quicker timeline depending on the auditing platform. Upon having a good grasp of your auditing speed, it may assist in selecting competitive audits that align with your capabilities, or even ones that push you to accelerate your pace.

## The Tincho Auditing Method

To illustrate the Tincho auditing method, we're going to refer to a video where Tincho performs a live auditing of the Ethereum Name Service (ENS).

> "I don't have a super formal auditing process. I will just show you briefly some things that I do..." - Tincho

### First Step

First thing's first - download the code, and **read the documentation**. You need to familiarize yourself with the content and context of the codebase, learn the jargon you can expect to see in the code and become comfortable with what the protocol is expected to do.

**READ THE DOCUMENTATION**

### Tools and Frameworks

Tincho describes a number of tools he uses while performing security reviews, bring the tools you're most familiar and best with.

- **VS Codeium**: a text editor with a privacy focus. It's based on VS Code but removes a lot of the user tracking telemetry
- **Foundry**: As a framework for reviewing codebases Foundry is incredibly fast and allows for quick testing with it's robust test suite
- **CLOC**: A simple command-line utility that helps count lines of code which can give a sense of the complexity of different parts of the codebase.
- **Solidity Metric**: Another tool developed by Consensys that provides useful metrics about your Solidity codebase.

By leveraging `CLOC` and `Solidity Metrics`, a security researcher can organize the codebase by complexity and systemically go through the contracts - marking them each complete as appropriate. This pragmatic approach ensures no stone is left unturned.

It's recommended to start with the smaller and more manageable contracts and build upon them as you go.

There's a point in an audit where your frame of mind should switch to an adversarial one. You should be thinking *"How can I break this..."*

![tincho1](SC-Security-note.assets/tincho1.png)

Given even simple functions like above, we should be asking ourselves

- **"Will this work for every type of token?"**
- **"Have they implemented access control modifiers properly?"**

> *USDT is a 'weird ERC20' in that it doesn't return a boolean on transferFrom calls*

### Audit, Review, Audit, Repeat

Keeping a record of your work is crucial in this process.

> Tincho recommends taking notes directly in the code *and* maintaining a separate file for raw notes/ideas.

Remember, there is always a risk of diving too deep into just one part of the code and losing the big picture. So, remember to pop back up and keep an eye on the over-all review of the code base.

Not everything you'll be doing is a manual review. Applying your knowledge of writing tests to verify suspicions is incredibly valuable. Tincho applies a `fuzz test` to his assessment of functions within the ENS codebase.

### Communication

Tincho describes keeping an open line of communication with the client/protocol as `fundamental`. The protocol is going to possess far more contextual understanding of what constitutes intended behavior than you will. Use them as collaborators. **`Trust but validate.`**

> "I would advise to keep the clients at hand. Ask questions, but also be detached enough." - Tincho

### Wrapping it Up

Sometimes it can feel like there's no end to the approaches you can make to a codebase, no end to the lines of code you can check and verify.

Tincho advocates for time-bounding yourself. Set limits and be as thorough as possible within them.

> "The thing is...I always get the feeling that you can be looking at a system forever." - Tincho

### The Audit Report and Follow Up

The last stage of this whole process is to present an audit report to the client. It should be clear and concise in the detailing of discovered vulnerabilities and provide recommendations on mitigation.

It's our responsibility as security researchers to review the implementation of any mitigations the client employs and to assure that *new bugs* aren't introduced.

### Aftermath of a Missed Vulnerability

There will always be the fear of missing out on some vulnerabilities and instead of worrying about things that slip through the net, aim to bring value beyond just identifying vulnerabilities. Be that collaborative security partner/educator the protocol needs to employ best practices and be prepared holistically.

As an auditor it's important to remember that you do not shoulder the whole blame when exploits happen. You share this responsibility with the client.

> This doesn't give you free reign to suck at your job. People will notice.

A last takeaway from Tincho:

> "Knowing that you’re doing your best in that, knowing that you’re putting your best effort every day, growing your skills, learning grows an intuition and experience in you."

## Reconnaissance

### Recon: Context



### Recon: Understanding the Code



## Exploit

### Exploit: Accsess Control



### Exploit: Public Data







# Puppy Raffle





# TSwap





# Thunder Loan





# Boss Bridge





# MEV & Governance