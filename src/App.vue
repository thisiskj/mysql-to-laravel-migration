<template>
    <div class="flex flex-col items-stretch min-h-screen">
        <div class="flex justify-between border-b-2 border-blue-500 py-4 px-8">
            <div>
                <div class="text-xl mb-1">
                    MySQL to Laravel Migration
                    <span class="text-sm ml-2 text-blue-500">BETA</span>
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
            <div class="flex-1">
                <textarea v-model="sql" class="p-4 text-sm font-mono w-full h-full focus:outline-none"></textarea>
            </div>
            <div class="border-l-2 border-blue-500"></div>
            <div class="flex-1">
        <textarea disabled v-model="migration" class="p-4 text-sm font-mono w-full h-full"></textarea>
            </div>
        </div>

        <div>
            <div v-if="error" class="border-2 rounded border-red-200 bg-red-50 text-red-500 p-2">
                Line {{ error.location.start.line }}:{{ error.location.start.column }} -
                {{ error.message }}
            </div>
            <div v-else class="border-2 rounded border-green-200 bg-green-50 text-green-500 p-2">
                Syntax Ok
            </div>
        </div>
    </div>
</template>

<script>
import NodeSQLParser from 'node-sql-parser'

export default {
    data() {
        return {
            sql: `CREATE TABLE users (
    id         int           not null auto_increment primary key,
    name       varchar(30)   null,
    is_admin   tinyint       not null default 0,
    bio        text          null comment='biography',
    record_key bigint        null,
    ratio      decimal(9, 4) null,
    created_at datetime      null,
    updated_at datetime      null
)
`,
            ast: null,
            error: null
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
                console.log(ast);
            } catch (error) {
                console.error(error);
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
            let method;
            let length;
            let columnName = def.column.column;

            switch (def.definition.dataType) {
                case "INT":
                    method = `integer('${columnName}')`;
                    break;

                case "TEXT":
                    method = `text('${columnName}')`;
                    break;

                case "VARCHAR":
                    length =
                        typeof def.definition.length !== "undefined"
                            ? def.definition.length
                            : 64;
                    method = `string('${columnName}', ${length})`;
                    break;

                case "CHAR":
                    length =
                        typeof def.definition.length !== "undefined"
                            ? def.definition.length
                            : 64;
                    method = `char('${columnName}', ${length})`;
                    break;

                case "TINYINT":
                    method = `tinyInteger('${columnName}')`;
                    break;

                case "SMALLINT":
                    method = `smallInteger('${columnName}')`;
                    break;

                case "BIGINT":
                    method = `bigInteger('${columnName}')`;
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
                let columnName = def.column.column;

                if (def.unique_or_primary === "primary key") {
                    indexes.push(`            $table->primary('${columnName}');`);
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
    }
};
</script>

<style></style>
