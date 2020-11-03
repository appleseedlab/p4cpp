|  Grammar Construct   |                                    CPP usage                                    |
|:--------------------:|:-------------------------------------------------------------------------------|
|   ParserDeclaration  |                        [Insert ParserStates using ifdefs](parser.md#1.1)                         |
|      ParserStates    | <ul><li> Leave it empty (barebones)</li> <li>Insert [parserStatements and/or stateExpression](state.md#1.2) based on defined flags</li><li>Also add [new selectCaseList](state.md#2.1) inside selectExpression blocks</li></ul> |
|  selectExpression  | [Insert new selectCase]((state.md#2.1)) statements based on defined flags |
|  controlDeclaration  | <ul><li>Insert new blockStatement (emit calls in parser files) inside the [apply controlBody section](control_apply.md#1.1)</li><ul><li>Insert [new table and/or action blocks](control_apply.md#3) based on defined flags/features</li><li> Add [new conditional statements](control_apply.md#3)</li><li>Insert [new controlLocalDeclarations](control_apply.md#1.2) based on flags</li><li>Adding [new instantiation](control_apple.md#7) based on flag</li></ul></ul> |
|  headerTypeDeclaration  | <ul><li>[headerTypeDeclaration itself is added](header_struct.md#2.1) to a program based on flags</li><ul><li> headerTypeDeclaration's structField are not added through CPPs</li></ul></ul> |
|  structTypeDeclaration  | <ul><li>[structTypeDeclaration itself is added](header_struct.md#1.2) based on defined flags</li><li>[structFields are added](header_struct.md#1.3) through ifdefs inside structTypeDeclaration blocks </li></ul> |
|  actionList  | Adding [new actionRef values](table.md#1.1) inside a table's action block |
|  actionDeclaration  | Adding [new blockStatement values](action.md#1.1) based on the flags |
