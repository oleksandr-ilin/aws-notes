Overview
Command-line interface for AWS services used for scripting and automation.

Tasks
- Install CLI, configure profiles/credentials, and script ops for automation.

Alternatives
| Alternative | Main pro | Main con |
|---|---:|---|
| SDKs (Python/JS) | Programmatic control | Requires code

Limitations
- Long scripts can be brittle; consider SDKs for complex logic.

Price info
- Free; costs are for the AWS API actions executed.

Network & multi-region
- Global tool; commands target regions per-resource or via explicit flags.

Popular use cases
- Automation scripts, ad-hoc queries, and bootstrap tasks.

When Not to Use
- Complex orchestration better handled by SDKs or IaC.

DR strategy
- Keep scripts in version control and store credentials securely (use roles instead of long-lived keys).
