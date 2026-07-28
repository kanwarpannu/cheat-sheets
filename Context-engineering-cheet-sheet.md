The reason for large context causing issues is:  
 - Hallucinations will eventually enter into it
 - Clashes in logic between multiple lines
 - Too much context causes agents to sidetrack or negatively influences over its training data

 After 30 plus tools agent performance starts degrading, around 100 typically it colapses. So implementing RAG over tool selection can have a lot better results.

Coding agent Prompting suggestions:
1. Start by saying `I want to build this new feature can you explore code base and suggest 2-3 different ways to build it before you start making any changes`.  
2. Then select an approach and say `create a plan before you start editing files`.  
3. End every prompt by saying `ask me as manay questions as needed`.  

Common workflows to flow with coding agents:
1. Explore -> Plan -> Confirm -> Code -> Commit (used for most common coding tasks)
2. Write Tests -> Commit -> Code -> Iterate -> Commit (TDD approach know to be great for backend dev)
3. Write Code -> Screenshot Result -> Iterate (for front end coding, give it a tool for screeshots to get great feedback)

When starting on a codebase:
1. init a MD file (eg. Claude.MD) in root level. This root level file is always loaded into context.
2. This file will have coding style, architecture of project/repo. Keep this file as small as possible as its loaded into context everytime.
3. every internal folder( eg. maven multi module) can have its own claude.MD file which is loaded into context once coding agent enters that folder.
4. Claude.MD files can be checked-in into the git. However we can create claude.local.md files if we want something for local only, this can skip git check-in.
5. Similarly for permissions on project level put them in .claude/settings.json and check-in this code. For local you can add to .claude/settings.local.json and this does not need to be check-in to git.
6. for project level MCP servers add to .mpc.json files.

Other suggestions:
1. Use different git worktrees if using multiple coding agents on same repos. 
2. Ask coding agent to suggest commit message. Its great at it.

TODO:
 Handling million lines of code ???
 AST parsing?
 the above stuff needs to have cursor ai equivalents as well. ????

 how to init instruction/claude.md files etc