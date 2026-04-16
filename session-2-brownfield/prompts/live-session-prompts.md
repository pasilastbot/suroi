/create-subagent create documentation-agent that follows the documentation guidelines at AGENTS.md

Please start iterative, paraller process of documenting the whole codebase according @AGENTS.md documentation plan. Start from tier 1 docs and once tier is done, dive into next level. Do comprehemsive work, do not lose details, do not hallucinate anything that isn't based on the codebase or existing docs. Use @.github/agents/documentation-agent.md for running in paraller.

It does not look like tier1 and tier2 are complete. Please refine and extend until they are 100% recognied.

Are you sure you have found all relevant subsystems?

Complete now full tier1 and 2 docs until you are happy.

@documentation-agent.agent.md complete the tier 3 documentation.

Switch to better model opus 4.5:

Please create a markdown file at docs/devquestions and list there top 50 questions that new senior developer might have about the codebase when starting as a new employee in the team. Do from multiple angles. Do not yet answer the questions.

Open new chat window

/create-subagent create qa answering agent that utilizes the documentation and documentation query / navigation at AGENTS.md

Switch to lighter model like haiku:

Please run 50 separate instances of qa-agent.md to answer each question at devquestions.md, run 5 in paraller. Write each answer under docs/qa folder as own markdown document.

Now evaluate each answer separately, how well did the documentation have answer to that question? Write documentation improvement proposals for each of them. Collect the reports at docs/qa-test-report.md.

Ok, now run all the improvements to the documentation so that we have 5/5

SPEC-DRIVEN DEVELOPMENT

Run the game, stop existing apps at port 3000 (haiku)

Open new chat window

I want to develop a new feature for this game:
- Single player mission where I have all the possible weapons etc from the beginning.
It should be the default mode that I spawn, I want to test all the weapons locally.
Use spec workflow for this.
Opus 4.5

This sounds good, please first plan the tasks at the specs folder before implementing.

Please start implementing. (vaihda malli sonnettiin)

Please add something to the game to shoot to.

Ok. please now have full review and audit for the changes we did.

Switch to another model like codex 5.3:
Please have another critical review and audit of the changes claude did.
