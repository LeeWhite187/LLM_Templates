# LLM_Templates
Set of templates for Interacting with LLMs

Preferences_Instructions.md
This is a simple prompt for the LLM, to ensure that it follows our design methodology when in design conversations.
And, that it follows our spec template, when generating output.
The content of this file gets pasted into the User Setting / Instructions for Claude text box.

METHODOLOGY.md
This is the primary design methodology document to be shared with the LLM, when doing design discussions.
It will shape the conversation into a design-first format.

SPEC_TEMPLATE_R1.md
This was the first version of the LLM spec template.
This is the template that the LLM will follow when creating requirements specs for a project.
It includes enough sections and details to harmonize the format and shape of spec documents across projects.

SPEC_TEMPLATE_R2.md
This template version included some updates to the title block, revision log, and removed the embedded conventions section from a spec document.

In the workinstruction folder, we store work instructions for migrating documents as templates an methodologies evolve.
Each is listed below:

Spec_Migration_Instruction_R1_to_R2.md
This is a work instruction for migrating any specs created with spec template R1 (or the unversioned spec template, in use prior to 2026-0601-0106).
Specifically, this work instruction makes clerical changes to the spec titleblock, revision history, and embedded convention section, to align with current formatting needs.
To use it, follow these steps:
1. Start a fresh chat with Claude.
2. Paste this work instruction as the first message.
3. Paste (or upload) the spec to be retrofitted.
4. Save the resulting retrofitted spec back over the original.
