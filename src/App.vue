<template>
    <div class="flex flex-col items-stretch min-h-screen">
        <div class="flex justify-between shadow border-b-2 border-gray-200 py-4 px-8">
            <div>
                <div class="text-xl mb-1">
                    MySQL to Laravel Migration
                </div>
                <div class="text-sm text-gray-500">
                    Generate migration files from an existing schema. To begin, enter the
                    "CREATE TABLE..." statement.
                </div>
            </div>
            <div>
                <a href="https://github.com/thisiskj/mysql-to-laravel-migration">
                    <img src="./assets/GitHub-Mark-32px.png" alt="Github"/>
                </a>
            </div>
        </div>

        <div class="flex-1 flex justify-between">
            <div class="flex-1 flex flex-col">
                <div class="bg-blue-500 text-white text-sm shadow px-2 py-4">
                    Step 1: Enter a CREATE TABLE schema below...
                </div>
                <div id="editor" class="flex-1 pt-2 w-full h-full"></div>
            </div>
            <div class="border-l-2 border-blue-500"></div>
            <div class="flex-1 flex flex-col bg-gray-200">
                <div class="bg-green-500 text-white text-sm shadow px-2 py-4">
                    Step 2: View the generated Laravel migration file below
                </div>
                <textarea disabled v-model="migration" class="p-4 text-sm font-mono w-full h-full"></textarea>
            </div>
        </div>

        <div v-if="error" class="fixed top-0 left-0 right-0 inset-2">
            <div class="border-2 border-red-700 bg-red-500 text-white p-2">
                Line {{ error.location.start.line }}:{{ error.location.start.column }} -
                {{ error.message }}
            </div>
        </div>
    </div>
</template>

<script>
import NodeSQLParser from 'node-sql-parser'
import * as monaco from 'monaco-editor'
import editorWorker from 'monaco-editor/esm/vs/editor/editor.worker?worker'
import jsonWorker from 'monaco-editor/esm/vs/language/json/json.worker?worker'
import cssWorker from 'monaco-editor/esm/vs/language/css/css.worker?worker'
import htmlWorker from 'monaco-editor/esm/vs/language/html/html.worker?worker'
import tsWorker from 'monaco-editor/esm/vs/language/typescript/ts.worker?worker'

self.MonacoEnvironment = {
    getWorker(_, label) {
        if (label === 'json') {
            return new jsonWorker()
        }
        if (label === 'css' || label === 'scss' || label === 'less') {
            return new cssWorker()
        }
        if (label === 'html' || label === 'handlebars' || label === 'razor') {
            return new htmlWorker()
        }
        if (label === 'typescript' || label === 'javascript') {
            return new tsWorker()
        }
        return new editorWorker()
    }
}

let editor = {};

export default {
    data() {
        return {
            sql: `CREATE TABLE users
(
    id         int     not null auto_increment primary key,
    name       varchar(30) null,
    is_admin   tinyint not null default 0,
    bio        text null comment='biography',
    record_key bigint null,
    ratio      decimal(9, 4) null,
    created_at datetime null,
    updated_at datetime null
)
`,
            ast: null,
            error: null,
            editorDecorations: [],
        };
    },
    methods: {
        parse() {
            try {
                this.error = null;
                this.ast = null;
                const parser = new NodeSQLParser.Parser();
                const ast = parser.astify(this.sql);

                if (Array.isArray(ast)) {
                    this.ast = ast[0];
                } else {
                    this.ast = ast;
                }

                if (editor && editor.deltaDecorations) {
                    editor.deltaDecorations(this.editorDecorations, []);
                }

                console.log(ast);
            } catch (error) {
                console.error(error);

                if (editor.deltaDecorations && error && error.location) {
                    this.editorDecorations = editor.deltaDecorations(this.editorDecorations, [
                        {
                            range: new monaco.Range(error.location.start.line, error.location.start.column, error.location.start.line, 9999),
                            options: {inlineClassName: 'editorInlineError'}
                        },
                    ]);
                }

                this.error = error;
            }
        },
        camelCase(s) {
            return s.replace(/([-_][a-z])/gi, ($1) => {
                return $1.toUpperCase().replace("-", "").replace("_", "");
            });
        },
        capitalize(s) {
            if (typeof s !== "string") {
                return "";
            }
            return s.charAt(0).toUpperCase() + s.slice(1);
        },
        methodForDef(def) {
            if (!def || !def.column) {
                return null;
            }

            let method;
            let length;
            let columnName = def.column.column;

            switch (def.definition.dataType) {
                case "BIGINT":
                    method = `bigInteger('${columnName}')`;
                    break;

                case "CHAR":
                    length =
                        typeof def.definition.length !== "undefined"
                            ? def.definition.length
                            : 64;
                    method = `char('${columnName}', ${length})`;
                    break;

                case "DATE":
                    method = `date('${columnName}')`;
                    break;

                case "DATETIME":
                    method = `dateTime('${columnName}')`;
                    break;

                case "DECIMAL":
                    length = def.definition.length;
                    let scale = def.definition.scale;

                    if (length && scale) {
                        method = `decimal('${columnName}', ${length}, ${scale})`;
                    } else if (length) {
                        method = `decimal('${columnName}', ${length})`;
                    } else {
                        method = `decimal('${columnName}')`;
                    }
                    break;

                case "JSON":
                    method = `json('${columnName}')`;
                    break;

                case "INT":
                    method = `integer('${columnName}')`;
                    break;

                case "SMALLINT":
                    method = `smallInteger('${columnName}')`;
                    break;

                case "TEXT":
                    method = `text('${columnName}')`;
                    break;

                case "TIMESTAMP":
                    method = `timestamp('${columnName}')`;
                    break;

                case "TINYINT":
                    method = `tinyInteger('${columnName}')`;
                    break;

                case "VARCHAR":
                    length =
                        typeof def.definition.length !== "undefined"
                            ? def.definition.length
                            : 64;
                    method = `string('${columnName}', ${length})`;
                    break;

                default:
                    method = `UNKNOWN_DATA_TYPE('${columnName}')`;
            }
            return method;
        },
        modifiersForDef(def) {
            let modifiers = [];

            if (def.nullable && def.nullable.type === "null") {
                modifiers.push("->nullable()");
            }

            if (def.auto_increment === "auto_increment") {
                modifiers.push("->autoIncrement()");
            }

            if (def.comment) {
                modifiers.push(`->comment('${def.comment.value.value}')`);
            }

            if (def.default_val) {
                modifiers.push(`->default(${def.default_val.value.value})`);
            }

            return modifiers.join("");
        },
        indexesFromAst() {
            let indexes = [];
            this.ast.create_definitions.forEach((def) => {
                if (def.column && def.column.column) {
                    let columnName = def.column.column;

                    if (def.unique_or_primary === "primary key") {
                        indexes.push(`            $table->primary('${columnName}');`);
                    }
                }

                if (def.constraint_type === "primary key") {
                    let columns = def.definition.map(c => `'${c}'`).join(',')
                    indexes.push(`            $table->primary(${columns});`);
                }

            });

            return indexes;
        }
    },
    computed: {
        migration() {
            if (!this.ast || this.error) {
                return "";
            }

            let tableName = this.ast.table[0].table;
            let tableNameCamelCase = this.capitalize(this.camelCase(tableName));

            let columns = this.ast.create_definitions
                .map((def) => {
                    let method = this.methodForDef(def);
                    let modifiers = this.modifiersForDef(def);

                    return `            $table->${method}${modifiers};`;
                })
                .join("\n");

            let indexes = this.indexesFromAst();

            return `<?php

use Illuminate\\Database\\Migrations\\Migration;
use Illuminate\\Database\\Schema\\Blueprint;
use Illuminate\\Support\\Facades\\Schema;

class Create${tableNameCamelCase}Table extends Migration
{
    /**
     * Run the migrations.
     *
     * @return void
     */
    public function up()
    {
        Schema::create('${tableName}', function (Blueprint $table) {
${columns}

            // Indexes
${indexes}
        });
    }

    /**
     * Reverse the migrations.
     *
     * @return void
     */
    public function down()
    {
        Schema::dropIfExists('${tableName}');
    }
}
`;
        }
    },
    watch: {
        sql() {
            this.parse();
        }
    },
    mounted() {
        this.parse();

        editor = monaco.editor.create(document.getElementById('editor'), {
            fontSize: "14px",
            value: this.sql,
            language: 'sql',
            minimap: {
                enabled: false
            },
        });

        editor.getModel().onDidChangeContent((event) => {
            console.log(editor.getValue())
            this.sql = editor.getValue()
        });
    },
};
</script>

<style>
.editorInlineError {
    @apply bg-red-500 text-white
}
</style>
