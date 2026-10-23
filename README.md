# MLPipe — A Compiler for End-to-End ML Pipelines

> Compiler Design course project. Write an ML workflow in a small DSL and compile it through every classic compiler phase: lexing, parsing, semantic analysis, IR optimization, code generation and execution.

**Live demo:** https://YOUR-USERNAME.github.io/YOUR-REPO/

## Why this project
ML workflows are usually hand-written, framework-specific scripts. Mistakes such as wrong stage order, repeated transformations or data leakage surface late, at run time. MLPipe treats the workflow as a *program*: it is validated early, optimized as an IR, and then turned into code.

## Compiler pipeline
```
Source DSL -> Lexer -> Parser -> AST -> Semantic Analysis -> IR Optimizer -> Code Generator -> Execution
              tokens   syntax   tree     validation         rewrites         scikit-learn      metrics
```

## The language
```
pipeline churn_prediction {
    load data "customer.csv";
    preprocess { fill_missing age; scale income; encode city; }
    split train 80 test 20;
    model random_forest { trees = 100; depth = 6; }
    train;
    evaluate metrics [accuracy, precision, recall, f1];
}
```

### Grammar
```
program    ::= pipeline
pipeline   ::= 'pipeline' ID '{' stmt* '}'
stmt       ::= load | preprocess | split | model | 'train' ';' | evaluate
load       ::= 'load' 'data' STRING ';'
preprocess ::= 'preprocess' '{' step* '}'
step       ::= ('fill_missing' | 'scale' | 'encode') ID ';'
split      ::= 'split' 'train' NUM 'test' NUM ';'
model      ::= 'model' ID '{' (ID '=' NUM ';')* '}'
evaluate   ::= 'evaluate' 'metrics' '[' ID (',' ID)* ']' ';'
```

## Phases in detail
| Phase | What it does |
|---|---|
| Lexer | Six token classes (KEYWORD, IDENTIFIER, STRING, NUMBER, SYMBOL, OPERATOR), line/column tracking, `//` comments, errors for illegal characters and unterminated strings. |
| Parser | Recursive descent, one function per grammar rule, typed AST nodes carrying source lines. Reports the first syntax error with line, column and a caret. |
| Semantic analysis | Six rule groups: stage ordering and required statements; columns exist and operations match types (scale numeric, encode categorical); split sums to 100 with both parts positive; known model and valid hyperparameters; supported metrics. Also emits warnings (unfilled missing values, dropped categorical columns, unscaled inputs for kNN/logistic regression). |
| IR optimizer | Block fusion (many `preprocess` blocks become one), redundant-step elimination (duplicates removed), canonical ordering (`fill_missing`, then `scale`, then `encode`). |
| Code generation | Emits a scikit-learn script using `ColumnTransformer` and `Pipeline`, so preprocessing is fitted on the training split only. |
| Execution | Runs the IR in the browser on a built-in synthetic customer dataset (500 rows, seeded) and reports the requested metrics and a confusion matrix. |

Supported models: `random_forest` (trees, depth), `decision_tree` (depth), `logistic_regression` (epochs), `knn` (k).
Supported metrics: `accuracy`, `precision`, `recall`, `f1`.

## Repository layout
```
index.html          website containing the complete compiler (JavaScript, no dependencies)
python-frontend/    Python lexer, parser, AST and unit tests (earlier front-end version)
README.md
```

## Running it
- **Website:** open `index.html` in a browser, or enable GitHub Pages (Settings -> Pages -> Deploy from branch -> `main` / root).
- **Python front end:**
  ```bash
  cd python-frontend/src
  python3 main.py ../samples/valid_churn.pipeline
  python3 -m unittest discover -s ../tests -v    # 17 tests
  ```

## Demo programs (in the website dropdown)
1. Churn prediction: all phases pass.
2. Optimizer demo: two preprocess blocks with duplicates and wrong order are fused, deduplicated and reordered.
3. kNN without scaling: compiles with semantic warnings.
4. Missing semicolon: parser error with line and column.
5. Semantic errors: ordering, type, column, split, model and metric errors reported together.

## Limitations
- Runs on one built-in dataset; the filename in `load data` is not read.
- One pipeline per program and a single target column (`churn`).
- The in-browser models are simplified implementations; the generated Python is what uses scikit-learn.

## Future work
CSV upload with schema inference, more models, cross-validation and hyperparameter search, a richer type system, export to Jupyter notebooks.

## Team
Four members: DSL and lexer; grammar and parser; AST, semantic analysis and optimizer; code generation, testing and website.
