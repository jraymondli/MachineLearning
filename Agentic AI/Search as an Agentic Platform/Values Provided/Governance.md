# Is it true that for LLM to enforce policy, that may not be 100% enforcement?



Yes. That is an important distinction, and it actually strengthens the governance argument.

If a policy is enforced only through **LLM instructions**—for example, “never retrieve salary data unless the user is HR”—you generally should **not treat that as 100% enforcement**. LLM behavior is probabilistic and can be affected by ambiguous instructions, prompt injection, conflicting context, tool descriptions, or simply model error.

For security- or compliance-critical policies, enforcement should therefore happen **outside the LLM**, in deterministic system components such as authorization services, tool gateways, retrieval filters, or policy engines. The LLM can participate in interpreting intent, but it should not be the final enforcement boundary.

So I would sharpen your wording to:

> **Governance:** enforce identity-, data-, and action-level policies deterministically at the platform and tool layers, rather than relying on probabilistic LLM behavior for policy enforcement.

Or, if you want to explicitly highlight the value over a generic Claude Code baseline:

> **Governance:** provide deterministic enforcement of identity-, data-, and action-level policies outside the LLM. Prompt-level policy instructions alone cannot provide the same enforcement guarantees because LLM behavior is inherently probabilistic.

A useful architectural principle here is:

**LLM decides what it wants to do → policy layer decides what it is actually allowed to do.**

That is a much stronger and more defensible ASAP value proposition than simply saying the agent is “instructed” to follow enterprise policies.
