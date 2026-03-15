# What This Document Is — And What It Is Not

Vibe coding has attracted a wave of guides, best practices, and community discussion. The picture that emerges from reading across formal guides, engineering blogs, and developer communities is consistent — and consistently incomplete.

### What the Existing Guides Get Right

The most commonly recommended practices across formal guides are:

- **Prompt engineering**: be specific, layer context, break tasks into small increments, ask before implementing
- **Version control**: use git, commit frequently, revert when the AI breaks something
- **Test after every change**: don't let AI changes accumulate unreviewed
- **Documentation**: maintain READMEs and comments because you didn't write the code line by line
- **Security basics**: don't hardcode credentials, validate inputs, do code review before deployment
- **Simple tech stacks**: choose mature, well-documented technologies that AI tools understand

These are sound practices. None of them are wrong.

### What the Developer Community Is Actually Experiencing

The gap between formal best-practice guides and real developer experiences is significant. Across community forums, the same failure patterns appear independently, described by people who hit them without having read the same guides:

**The three-month wall.** Multiple developers independently describe hitting a point — typically around the three-month mark — where the codebase has grown beyond anyone's ability to understand it. One AI fix breaks ten other things. Red Hat Developer named this "the whack-a-mole effect": *"AI is still just soooooo stupid and it will fix one thing but destroy 10 other things in your code."*

**The unfixable mess.** Developers describe codebases becoming genuinely unrescuable — "functions inside functions, conditions triggering other conditions" — to the point of abandonment. Not because the AI was incompetent, but because there were no structural constraints on where things could go, and the accumulated decisions became incoherent.

**The context window trap.** The widely identified problem is that agents "forget" between sessions. The widely offered solution is to periodically reset sessions or maintain a context file. Neither addresses the root cause: without persistent structure encoded in the codebase itself — interfaces, module boundaries, documented ownership — each new session starts from ignorance.

**The command quality problem.** A less-discussed failure mode: when the human giving commands doesn't understand the architecture well enough to give architecturally correct instructions. The agent executes faithfully. The result is a feature built in the wrong place, which contradicts existing contracts, which requires a large refactor, which generates a PR too large to review properly. In a survey of 18 CTOs, 16 reported production disasters directly caused by AI-generated code. The AI wrote what it was asked. The commands were the problem.

### The Spectrum of Community Positions

The community has split into two camps, both of which miss something important:

**The anti-vibe-coding camp** (dominant in communities like r/programming and r/cscareerquestions) holds that vibe coding is fundamentally incompatible with professional engineering — a "red flag in interviews," evidence of undisciplined thinking, and incompatible with team collaboration. This position is wrong not because the criticisms are false, but because it treats the failure modes as inherent to AI-assisted development rather than as consequences of absent structure.

**The pro-vibe-coding camp** (dominant in builder and founder communities) holds that vibe coding is a superpower that democratizes software creation and makes traditional engineering concerns obsolete. This position is wrong because it has not yet experienced the three-month wall, or has not yet tried to hand the project to someone else, or has not yet needed to change something foundational.

The most accurate framing comes from Addy Osmani's distinction between "vibe coding" and "AI-assisted engineering": vibe coding is for throwaway experiments; AI-assisted engineering integrates AI into a structured development process with human oversight of architecture and review. This document extends that framing by providing the specific structural standards that make AI-assisted engineering concrete and actionable.

### What They All Miss

Every guide and community discussion operates at the level of workflow habits and session management. They describe how to interact with an AI tool more effectively within whatever structure you already have. None of them describe what structure to build in the first place.

Nobody names the architecture and explains why it's the right choice. Nobody defines interface contracts as the mechanism that closes architectural debates permanently. Nobody describes module isolation as the structural prevention of logic leakage and feature duplication. Nobody explains how PR reviews should feed back into automated enforcement rules. Nobody describes the development loop as a required sequence — spec first, interface second, tests third, implementation last — that determines the quality of everything after it.

**The result: guidance that helps you prompt better today, but leaves you unprepared for month three.**

A consistent pattern across all existing guidance: vibe coding is framed as appropriate for prototypes and MVPs, implying that production systems require returning to traditional methods. **This document rejects that framing.** Vibe coding is viable for production — when the right architectural standards are in place. The problem is not the AI. The problem is the absence of structure.

### What This Document Is

A set of numbered engineering standards covering:
1. Which architecture to start with and why
2. The exact module structure and the rationale for uniformity
3. The required development loop: Spec → Interface → Tests → Implementation
4. The difference between guidelines (advisory) and guardrails (structural enforcement)
5. The human role: planning, commanding correctly, and converting PR patterns into automated rules
6. How today's decisions enable microservice extraction later

### What This Document Is Not

- **A prompt engineering guide.** Prompting technique is a separate topic. This document addresses what the codebase looks like, not how you describe what you want.
- **Tool-specific.** These standards apply to any AI coding assistant — Cursor, Claude Code, Copilot, or others.
- **Anti-vibe-coding.** The goal is not to write everything by hand. The goal is to build a codebase where agents can work reliably at step 50, not just step 5.
- **A guide for throwaway demos.** If you are building a prototype that will be discarded in a week, most of this is overkill. This is for projects that need to grow.
- **Complete.** These standards reflect patterns proven effective in production systems. They are a starting point, not a ceiling.
