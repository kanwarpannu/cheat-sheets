The reason for large context causing issues is:  
 - Hallucinations will eventually enter into it
 - Clashes in logic between multiple lines
 - Too much context causes agents to sidetrack or negatively influences over its training data

 After 30 plus tools agent performance starts degrading, around 100 typically it colapses. So implementing RAG over tool selection can have a lot better results.


TODO:
 Handling million lines of code ???
 AST parsing?

 how to init instruction/claude.md files etc