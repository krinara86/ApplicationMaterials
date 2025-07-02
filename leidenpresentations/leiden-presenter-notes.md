# Detailed Presenter Notes for Leiden Slides

## Slide 1: Title Slide
 I am Krishna. Thanks a lot for inviting me to give the talk and organising this whole day. I am quite excited. 

## Slide 2: Agenda
Today, I'll take you through three interconnected parts of my research journey. First, we'll examine why API misuses represent an important challenge - one that can costs companies a lot and affects virtually every application. Then, I'll present domain-specific solutions I've worked on, including CrySL for static specification and jGuard for runtime prevention - showing how embedding domain expertise directly into APIs can eliminate entire classes of vulnerabilities. Finally, I'll share my vision for how these domain-specific modeling techniques can address the broader challenges we face with AI-generated code - essentially, how the lessons from API misuse prevention can guide us toward more reliable AI-assisted development

## Slide 3: API Misuse Example
By APi misuse I mean .... This is sort of a toned down informal definition, but this works for our scope of discussion now. Here is a use of javas standard inhouse crypto API to do. I have picked a piece of code from the security world because it drives the seriousness even more, but these sort of misuses happen in all sorts of APIs. ( File api,  Database connection apis, etc.) Essentially, This code shows a typical cryptographic setup in Java - generating a key, creating a cipher, and encrypting data. While it looks straightforward, there are multiple points where a developer can silently introduce critical security vulnerabilities

The autocomplete dropdown shows the overwhelming choices
"AES" alone defaults to insecure ECB mode
128-bit keys might be too weak for some standards
Missing initialization vectors
No guidance on secure random sources
Compiler gives zero warnings about any of this

## Slide 4: The $85 Million Problem
• Avoiding API misuses are not just a problem of helping the developer. These issues can end up costing a lot. 
• **Zoom story**: "Non-secure AES configuration exposed user data" and they had to settle for 85 million finally when it was pointed out it was because they were using the wrong algorithm. 


## Slide 5: Why Current Solutions Fall Short
What is available in the way of avoiding these issues as state of the art? API documentation. These are cumbersome, no one reads them. Static analysis has a bunch of issues itself.  


## Slide 6: Three Main API Misuse Patterns (Visual)
We saw what API misuses are. Before we look at the solutions, lets first narrow the scope of discussion. If we want domain specific solutions, we would need to define our domain. In the case of API misuses, that would mean coming up how mususes look in practice. Here are the three main misuse types as gathered by a large scale study on API misuses. 

## Slide 7: Three Main API Misuse Patterns (Code Examples)
• **Walk through each image**:
  - **Insecure Parameter**: Here the misuses are essentially wrongly initialized parameter whatever that means to that particular context of the API use. 'AES' without mode? Defaults to ECB - Same input gives same output - that's terrible for security. It's like if your password always produced the same scrambled version. Attackers can spot patterns and break it!"
  - **Incorrect Call**: These are misuses where you have an unexpected method call at a particular run of the code. For example, you would expect a file to be closed after it is open. In this case, "Missing update() - the data never gets into the signature!"
  - **Insecure Composition**: "64-bit keys in 2025 is consideres quite crackable. "
• **MuBench reference**: "89 real GitHub projects with these exact mistakes"
• **Transition**: "These all follow state machine patterns..."
• **Time**: 2 minutes

## Slide 8: API Usage as State Machines
• **Key insight**: "APIs have implicit state machines"
• **Connect to DSLs**: "What if we could make these states explicit?"
• **Example**: "Just like an Iterator - hasNext before next"

## Slide 9: CrySL Overview
• **Context**: "Before my work, CrySL was state-of-the-art" . It was a DSL to express API usage patterns (Essentially all the three types of misuses we saw) and there is a static analyser behind thet checks for compliance of the code with the DSL specifications. 
CrySL represents the current state-of-the-art in API misuse specification. Let me walk through its approach.
The left side defines the structural elements - objects and API events. For each method like getInstance or initSign, we enumerate all possible parameter combinations. The right side encodes the behavioral constraints and temporal properties.
In the CONSTRAINTS section, we restrict algorithm choices to secure options. The ORDER section specifies valid method sequences as a regular expression: getInstance, followed by optional initSign, then one or more updates, finally sign. This effectively encodes the state machine we saw earlier.
CrySL is comprehensive - 944 rules cover Java's cryptographic APIs. However, it operates as an external specification language, requiring developers to install CogniCrypt and execute separate analysis passes. The fundamental limitation is this separation between API implementation and safety specification.
This motivated my work on jGuard: what if we could embed these specifications directly into the API implementation? No external tooling, no separate analysis phase - just APIs that enforce their own correct usage

## Slide 10: The Challenge (Reiteration)
The challenge extends beyond security APIs. Consider the Iterator pattern - fundamental to Java, used millions of times daily.
Look at this code. Whether next() is safe depends entirely on what hasNext() returned. If it returned false, calling next() throws NoSuchElementException. This is a classic API misuse that crashes applications.
Here's the key insight: static analysis cannot determine the return value of hasNext() - it depends on runtime data like collection contents. No amount of sophisticated analysis can predict this statically. Yet this pattern causes real crashes in production systems.
This exemplifies why we need runtime enforcement. Some API constraints are inherently dynamic. Static analysis, no matter how advanced, has fundamental limitations. This motivates jGuard's approach of embedding runtime checks directly into APIs



## Slide 11: JGuard High-Level Overview
• **Core innovation**: "State machines as first-class citizens IN the API"
• **Walk through flow diagram**: 
  - "API call comes in"
  - "Check guards (our state variables)"
  - "Execute method"
  - "Update guards for next call"
• **Code example**: Point to verified class, guards, requirements
• **Key differentiator**: "No external tools needed!"
• **Time**: 2 minutes
"jGuard embeds state machines directly into API implementations. The left diagram shows the conceptual flow: method calls check guards, execute, then update guards. The right shows the actual syntax - verified classes with explicit guard variables and requirements. 
 these compile to regular Java. No external tools required. API users get safety guarantees transparently."

## Slide 12: JGuard Component 1 - Guards
"Guards make implicit API state explicit. On the left, the jGuard DSL declares boolean state variables. The 'finally false' annotation requires that the guard must be false (reset) when the object is destroyed. In this example, 'encryptionStarted finally false' means any encryption operation that was started must be completed (which resets the guard) before the object can be safely garbage collected. If the object is destroyed while encryptionStarted is still true, jGuard reports a misuse - indicating an incomplete encryption operation.
This compiles to standard Java fields with finalizer checks. The transformation is straightforward - guards become booleans, finally clauses generate finalizer validation. API users see normal Java, but get state machine enforcement."


## Slide 13: JGuard Component 2 - Requirements
"Requirements encode preconditions declaratively. The jGuard syntax specifies what must be true before method execution.
The compiler generates simple if-checks at method entry. Failed requirements trigger reportMisuse()."

## Slide 14: JGuard Component 3 - Consequences
"Consequences update guards based on method outcomes. The Iterator example demonstrates conditional state changes - hasNext returning true enables next().
The compilation strategy uses method wrapping. Original methods become private with mangled names. Public wrappers handle state updates based on return values. This ensures state changes cannot be forgotten or incorrectly implemented."

## Slide 15: Exception Handling
Real-world APIs must handle exceptional control flow. Consider what happens when encryption fails mid-operation - perhaps due to invalid data, hardware security module errors, or memory issues. Without proper state management, the Cipher object could be left in an inconsistent state - partially initialized, thinking it's ready for operations it cannot perform.
jGuard consequences support exception triggers to handle these scenarios. Look at the example: 'when throws CryptoException resets isInitialized'. This means if the encryption method throws this exception, we automatically reset the initialization state. The object returns to a safe, predictable state rather than remaining in limbo.
The generated code wraps the original method in try-catch blocks. On success, we update guards normally. On exception, we execute the exception-specific guard updates before re-throwing. This is critical - imagine a Cipher that fails during initialization but still thinks it's initialized. Subsequent operations would fail mysteriously or, worse, silently produce incorrect results.
This pattern prevents an entire class of bugs where partial operations leave objects in zombie states. In cryptographic APIs, this is especially dangerous - a partially initialized cipher might fall back to insecure defaults or leak sensitive data. By encoding exception handling into our state machine, we ensure consistent, predictable behavior even when things go wrong."

## Slide 16: Meta Variables
"API constraints vary by deployment context. Meta variables enable context-specific validation - FIPS allows TripleDES, BSI prohibits it.
Here's an elaborated version of the meta variables presenter note:
"Each instantiation generates a separate class with customized validation rules. Let me explain the compilation strategy and why this matters.
When you write 'instantiation FIPS' and 'instantiation BSI', the jGuard compiler doesn't create one class with runtime flags or configuration files. Instead, it generates two completely separate Java classes: Cipher_FIPS and Cipher_BSI. Each has its allowed algorithms hard-coded as static final sets.
This design has several advantages. First, compliance is verified at compile-time, not runtime. If you try to use TripleDES in Cipher_BSI, it fails immediately - no runtime surprises. Second, there's zero performance overhead from checking configuration flags or consulting external rule files. The validation is compiled directly into bytecode.
Most importantly, this supports regulatory compliance scenarios. A financial application shipping to Germany can include only Cipher_BSI.jar, physically unable to use algorithms that BSI prohibits. An application for US government use includes only Cipher_FIPS.jar. The compliance is baked into the artifact itself - auditors can verify compliance by examining which JAR is deployed.
This also enables gradual migration. An organization moving from FIPS to BSI standards can run both versions simultaneously during transition, with different components using different implementations. No global configuration changes, no runtime switches - just type-safe, compile-time guaranteed compliance.
The alternative - runtime configuration - is error-prone. One misconfigured property file could violate security standards. Our approach makes misconfiguration impossible: if it compiles and runs, it's compliant.


## Slide 17: Empirical Validation
"We evaluated jGuard on the MUBench dataset. Key results: 89.2% of real-world misuses expressible, zero false positives compared to CogniCrypt's 67, negligible performance overhead.The precision difference is fundamental - runtime checking has complete program state, while static analysis must approximate. Here's a concise elaboration:
"The precision difference is fundamental - runtime checking has complete program state, while static analysis must approximate.
Consider our ECB mode example. Static analysis sees Cipher.getInstance("AES") and must assume all possible values that string could have - it might be computed, loaded from config, or user input. So it warns about potential ECB usage even when the code might actually compute "AES/GCM/NoPadding" at runtime. This uncertainty causes false positives.
Runtime checking is different. When getInstance executes, we know the exact string value. No approximation, no uncertainty. If it's "AES", we catch it. If it's "AES/GCM/NoPadding", we don't. This is why jGuard achieves zero false positives while CogniCrypt reports 67.
The same applies to control flow. Static analysis must consider all possible paths through your code. Runtime checking only deals with the path actually taken. This fundamental difference - complete information versus conservative approximation - explains why runtime enforcement can be so much more precise."
Key point: Static analysis trades precision for early detection. jGuard trades early detection for perfect precision. For API misuse, precision matters more because false positives train developers to ignore warnings. Performance impact is minimal because checks are simple boolean tests."


## Slide 18: Transition to AI Vision
"jGuard demonstrates the value of embedding domain expertise directly into implementations. This principle extends beyond API safety.
The broader insight: explicit modeling of correctness properties enables systematic guarantees. This becomes critical when we consider AI-generated code."



## Slide 19: Promise of LLMs
"Large Language Models are transforming software development. Current capabilities include code generation, summarization, and debugging assistance.
The SE 3.0 vision proposes AI as an intelligent collaborator. However, this assumes LLMs can provide reliable, correct outputs."

The SE 3.0 vision comes from Hassan et al.'s position paper at FSE 2024, 'Towards AI-Native Software Engineering.' This represents a systematic attempt to characterize how software engineering might evolve with widespread LLM adoption.
The paper identifies three eras: SE 1.0 focused on manual programming, SE 2.0 introduced IDEs and tooling, and SE 3.0 envisions AI-native development. Their core proposal is 'intent-first, conversation-oriented development' - developers express requirements in natural language, AI systems handle implementation.
Key technical claims include: AI understanding ambiguous specifications, generating complete systems from high-level descriptions, and automatically adapting code to requirement changes. The framework assumes the communication burden shifts from human to AI - precise prompting becomes unnecessary as AI infers developer intent.
However, this assumes LLMs can provide reliable, correct outputs. Current empirical evidence shows significant gaps: hallucination rates up to 76.3% in some contexts, systematic generation of vulnerable code patterns, and inability to maintain consistency across large codebases. The paper acknowledges these as challenges but doesn't address whether they're fundamental limitations or engineering problems.
This motivates our hybrid approach. We accept the premise that AI will be central to future development, but argue that pure generation is insufficient. Systematic validation through static analysis and formal specifications is essential. The realistic path to SE 3.0 requires combining AI capabilities with correctness guarantees - exactly what our architecture provides.

## Slide 20: The Black Box Problem
LLMs have fundamental limitations rooted in their architecture. They operate through statistical pattern matching, not logical reasoning. Tie et al. (2024) demonstrate this empirically - LLMs fail at multi-step reasoning and abstract thinking, achieving only 23% accuracy on logical inference tasks.
Hallucinations are systematic. The same study shows 76.3% hallucination rates in complex scenarios. LLMs generate syntactically plausible but incorrect code because they match patterns without understanding correctness.
Context limitations compound these issues. With 8-128K token windows, LLMs cannot maintain consistency across large codebases. Each request processes in isolation, losing critical system-wide invariants.
Security implications are severe. Pearce et al. (2022) found 40% of GitHub Copilot's generated code contains security vulnerabilities. These aren't random - LLMs consistently replicate insecure patterns from training data: hardcoded passwords, SQL injections, weak cryptography.
The key insight: these are architectural properties, not bugs. Statistical models cannot distinguish secure from insecure when both appear frequently in training data. This is why Tie et al. conclude LLMs 'cannot be sole judges of correctness' - they lack the logical foundation required for security reasoning

## Slide 21: Why LLMs Fail at Program Comprehension
In our recent ACL Findings paper, we investigated how code LLMs actually process programs. We found they operate as sophisticated dictionary systems, not true comprehension engines.
The experiment was revealing. We tested whether LLMs could integrate information across syntactic and semantic domains. They excel at syntax-to-syntax associations - knowing that 'if' pairs with braces, that certain tokens co-occur. They're equally good at identifier patterns - recognizing that 'key' appears near 'encrypt'.
But here's the critical failure: they cannot connect syntax to program semantics. When you write Cipher.getInstance("AES"), the model knows this pattern appears frequently in training data. It doesn't understand that "AES" implies ECB mode, which enables pattern attacks.
This explains systematic API misuses. LLMs reproduce statistically common patterns without security comprehension. They've memorized that getInstance takes a string parameter, but not what that string means for security. This isn't a training problem - it's architectural. Dictionary learning cannot bridge the syntax-semantic gap.
For API safety, this is fatal. Security requires understanding consequences, not just matching patterns. Our findings demonstrate why hybrid approaches are essential - we need systems that can verify semantic correctness, not just syntactic plausibility

## Slide 22: The Expertise Gap

The TOMMY study, conducted by Richards and Wessel in 2024, investigated how LLM-based coding assistants could adapt to different users' mental states. The researchers built a system that infers developer expertise, knowledge gaps, and comprehension patterns - essentially giving the AI a 'Theory of Mind' about its users.
They tested TOMMY with developers ranging from novices to experts, examining how personalized explanations affected code understanding and task completion. The key innovation was treating developer mental state as a first-class concern in AI assistance.


The TOMMY study quantified expertise effects on LLM usage. Expert programmers provide better context and recognize errors. Novices struggle with both.
Current systems require users to compensate for LLM limitations. This creates unequal access to AI benefits. We need systems that work regardless of user expertise."


## Slide 23: Our Vision - Hybrid System
We propose combining LLM fluency with static analysis correctness through structured context representation.
The developer context DSL has two components: mental model capturing user expertise and intent, program context providing semantic information through ASTs and flow graphs. This design reflects how developers actually think about code - cognitive science research shows programmers maintain mental models of program state, control flow, and domain concepts while coding. We're simply making these implicit mental representations explicit and machine-readable.
The TOMMY system demonstrated this empirically - by inferring developer mental states like comprehension level and background knowledge, it achieved significantly better assistance outcomes. Our DSL formalizes what TOMMY discovered: effective AI assistance requires understanding not just the code, but the human examining it.
This structured representation enables precise reasoning and personalized assistance, overcoming natural language limitations. Instead of hoping an LLM infers context from vague prompts, we provide explicit, structured information about both the developer's understanding and the program's actual semantics.

## Slide 24: Case Study - ECB Misuse Prevention
Consider the complete flow: LLM generates insecure ECB mode, static analysis detects the vulnerability, system queries developer context for expertise level, generates personalized explanation, provides secure alternative.
The key is context-aware feedback. Novices get different explanations than experts. The system adapts to the user rather than requiring adaptatio

## Slide 25: Iterative Refinement
"The system improves through feedback loops. Static analysis findings train the LLM, user feedback refines mental model inference, property-based testing validates complex scenarios.
This creates a trust-but-verify architecture where LLM creativity combines with formal method correctness.

## Slide 27: Evidence from Literature
"Our approach is grounded in empirical findings. Tie et al. demonstrate LLM limitations in logical reasoning. The TOMMY study establishes the expertise gap. Hassan et al.'s SE 3.0 vision calls for exactly this kind of hybrid approach.
The innovation is our specific architecture: structured context representation with systematic validation

