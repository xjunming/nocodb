<script setup lang="ts">
import type { Ref } from 'vue'
import type { ListItem as AntListItem } from 'ant-design-vue'
import jsep from 'jsep'
import type { ColumnType } from 'nocodb-sdk'
import { UITypes, jsepCurlyHook } from 'nocodb-sdk'
import {
  MetaInj,
  NcAutocompleteTree,
  formulaList,
  formulaTypes,
  formulas,
  getUIDTIcon,
  getWordUntilCaret,
  insertAtCursor,
  onMounted,
  useColumnCreateStoreOrThrow,
  useDebounceFn,
  validateDateWithUnknownFormat,
} from '#imports'

interface Props {
  value: Record<string, any>
}

const props = defineProps<Props>()
const emit = defineEmits(['update:value'])
const vModel = useVModel(props, 'value', emit)

const { setAdditionalValidations, validateInfos, sqlUi, column } = useColumnCreateStoreOrThrow()

enum JSEPNode {
  COMPOUND = 'Compound',
  IDENTIFIER = 'Identifier',
  MEMBER_EXP = 'MemberExpression',
  LITERAL = 'Literal',
  THIS_EXP = 'ThisExpression',
  CALL_EXP = 'CallExpression',
  UNARY_EXP = 'UnaryExpression',
  BINARY_EXP = 'BinaryExpression',
  ARRAY_EXP = 'ArrayExpression',
}

const meta = inject(MetaInj)

const columns = computed(() => meta?.value?.columns || [])

const validators = {
  formula_raw: [
    {
      validator: (_: any, formula: any) => {
        return new Promise<void>((resolve, reject) => {
          const res = parseAndValidateFormula(formula)
          if (res !== true) {
            return reject(new Error(res))
          }
          resolve()
        })
      },
    },
  ],
}

const availableFunctions = formulaList

const availableBinOps = ['+', '-', '*', '/', '>', '<', '==', '<=', '>=', '!=']

const autocomplete = ref(false)

const formulaRef = ref()

const sugListRef = ref()

const sugOptionsRef = ref<typeof AntListItem[]>([])

const wordToComplete = ref<string | undefined>('')

const selected = ref(0)

const sortOrder: Record<string, number> = {
  column: 0,
  function: 1,
  op: 2,
}

const suggestionsList = computed(() => {
  const unsupportedFnList = sqlUi.value.getUnsupportedFnList()
  return [
    ...availableFunctions
      .filter((fn: string) => !unsupportedFnList.includes(fn))
      .map((fn: string) => ({
        text: `${fn}()`,
        type: 'function',
        description: formulas[fn].description,
        syntax: formulas[fn].syntax,
        examples: formulas[fn].examples,
      })),
    ...columns.value
      .filter((c: Record<string, any>) => {
        // skip system LTAR columns
        if (c.uidt === UITypes.LinkToAnotherRecord && c.system) return false
        // v1 logic? skip the current column
        if (!column) return true
        return column.value.id !== c.id
      })
      .map((c: any) => ({
        text: c.title,
        type: 'column',
        icon: getUIDTIcon(c.uidt),
        uidt: c.uidt,
      })),
    ...availableBinOps.map((op: string) => ({
      text: op,
      type: 'op',
    })),
  ]
})

// set default suggestion list
const suggestion: Ref<Record<string, any>[]> = ref(suggestionsList.value)

const acTree = computed(() => {
  const ref = new NcAutocompleteTree()
  for (const sug of suggestionsList.value) {
    ref.add(sug)
  }
  return ref
})

function parseAndValidateFormula(formula: string) {
  try {
    const parsedTree = jsep(formula)
    const metaErrors = validateAgainstMeta(parsedTree)
    if (metaErrors.size) {
      return [...metaErrors].join(', ')
    }
    return true
  } catch (e: any) {
    return e.message
  }
}

function validateAgainstMeta(parsedTree: any, errors = new Set(), typeErrors = new Set()) {
  if (parsedTree.type === JSEPNode.CALL_EXP) {
    // validate function name
    if (!availableFunctions.includes(parsedTree.callee.name)) {
      errors.add(`'${parsedTree.callee.name}' function is not available`)
    }
    // validate arguments
    const validation = formulas[parsedTree.callee.name] && formulas[parsedTree.callee.name].validation
    if (validation && validation.args) {
      if (validation.args.rqd !== undefined && validation.args.rqd !== parsedTree.arguments.length) {
        errors.add(`'${parsedTree.callee.name}' required ${validation.args.rqd} arguments`)
      } else if (validation.args.min !== undefined && validation.args.min > parsedTree.arguments.length) {
        errors.add(`'${parsedTree.callee.name}' required minimum ${validation.args.min} arguments`)
      } else if (validation.args.max !== undefined && validation.args.max < parsedTree.arguments.length) {
        errors.add(`'${parsedTree.callee.name}' required maximum ${validation.args.max} arguments`)
      }
    }
    parsedTree.arguments.map((arg: Record<string, any>) => validateAgainstMeta(arg, errors))

    // validate data type
    if (parsedTree.callee.type === JSEPNode.IDENTIFIER) {
      const expectedType = formulas[parsedTree.callee.name].type
      if (expectedType === formulaTypes.NUMERIC) {
        if (parsedTree.callee.name === 'WEEKDAY') {
          // parsedTree.arguments[0] = date
          validateAgainstType(
            parsedTree.arguments[0],
            formulaTypes.DATE,
            (v: any) => {
              if (!validateDateWithUnknownFormat(v)) {
                typeErrors.add('The first parameter of WEEKDAY() should have date value')
              }
            },
            typeErrors,
          )
          // parsedTree.arguments[1] = startDayOfWeek (optional)
          validateAgainstType(
            parsedTree.arguments[1],
            formulaTypes.STRING,
            (v: any) => {
              if (
                typeof v !== 'string' ||
                !['sunday', 'monday', 'tuesday', 'wednesday', 'thursday', 'friday', 'saturday'].includes(v.toLowerCase())
              ) {
                typeErrors.add(
                  'The second parameter of WEEKDAY() should have the value either "sunday", "monday", "tuesday", "wednesday", "thursday", "friday" or "saturday"',
                )
              }
            },
            typeErrors,
          )
        } else {
          parsedTree.arguments.map((arg: Record<string, any>) => validateAgainstType(arg, expectedType, null, typeErrors))
        }
      } else if (expectedType === formulaTypes.DATE) {
        if (parsedTree.callee.name === 'DATEADD') {
          // parsedTree.arguments[0] = date
          validateAgainstType(
            parsedTree.arguments[0],
            formulaTypes.DATE,
            (v: any) => {
              if (!validateDateWithUnknownFormat(v)) {
                typeErrors.add('The first parameter of DATEADD() should have date value')
              }
            },
            typeErrors,
          )
          // parsedTree.arguments[1] = numeric
          validateAgainstType(
            parsedTree.arguments[1],
            formulaTypes.NUMERIC,
            (v: any) => {
              if (typeof v !== 'number') {
                typeErrors.add('The second parameter of DATEADD() should have numeric value')
              }
            },
            typeErrors,
          )
          // parsedTree.arguments[2] = ["day" | "week" | "month" | "year"]
          validateAgainstType(
            parsedTree.arguments[2],
            formulaTypes.STRING,
            (v: any) => {
              if (!['day', 'week', 'month', 'year'].includes(v)) {
                typeErrors.add('The third parameter of DATEADD() should have the value either "day", "week", "month" or "year"')
              }
            },
            typeErrors,
          )
        }
      }
    }

    errors = new Set([...errors, ...typeErrors])
  } else if (parsedTree.type === JSEPNode.IDENTIFIER) {
    if (
      columns.value
        .filter((c: Record<string, any>) => !column || column.value.id !== c.id)
        .every((c: Record<string, any>) => c.title !== parsedTree.name)
    ) {
      errors.add(`Column '${parsedTree.name}' is not available`)
    }

    // check circular reference
    // e.g. formula1 -> formula2 -> formula1 should return circular reference error

    // get all formula columns excluding itself
    const formulaPaths = columns.value
      .filter((c: Record<string, any>) => c.id !== column?.value.id && c.uidt === UITypes.Formula)
      .reduce((res: Record<string, any>[], c: Record<string, any>) => {
        // in `formula`, get all the target neighbours
        // i.e. all column id (e.g. cl_xxxxxxxxxxxxxx) with formula type
        const neighbours = (c.colOptions.formula.match(/cl_\w{14}/g) || []).filter(
          (colId: string) => columns.value.filter((col: ColumnType) => col.id === colId && col.uidt === UITypes.Formula).length,
        )
        if (neighbours.length > 0) {
          // e.g. formula column 1 -> [formula column 2, formula column3]
          res.push({ [c.id]: neighbours })
        }
        return res
      }, [])
    // include target formula column (i.e. the one to be saved if applicable)
    const targetFormulaCol = columns.value.find((c: ColumnType) => c.title === parsedTree.name && c.uidt === UITypes.Formula)

    if (targetFormulaCol && column?.value.id) {
      formulaPaths.push({
        [column?.value?.id as string]: [targetFormulaCol.id],
      })
    }
    const vertices = formulaPaths.length
    if (vertices > 0) {
      // perform kahn's algo for cycle detection
      const adj = new Map()
      const inDegrees = new Map()
      // init adjacency list & indegree

      for (const [_, v] of Object.entries(formulaPaths)) {
        const src = Object.keys(v)[0]
        const neighbours = v[src]
        inDegrees.set(src, inDegrees.get(src) || 0)
        for (const neighbour of neighbours) {
          adj.set(src, (adj.get(src) || new Set()).add(neighbour))
          inDegrees.set(neighbour, (inDegrees.get(neighbour) || 0) + 1)
        }
      }
      const queue: string[] = []
      // put all vertices with in-degree = 0 (i.e. no incoming edges) to queue
      inDegrees.forEach((inDegree, col) => {
        if (inDegree === 0) {
          // in-degree = 0 means we start traversing from this node
          queue.push(col)
        }
      })
      // init count of visited vertices
      let visited = 0
      // BFS
      while (queue.length !== 0) {
        // remove a vertex from the queue
        const src = queue.shift()
        // if this node has neighbours, increase visited by 1
        const neighbours = adj.get(src) || new Set()
        if (neighbours.size > 0) {
          visited += 1
        }
        // iterate each neighbouring nodes
        neighbours.forEach((neighbour: string) => {
          // decrease in-degree of its neighbours by 1
          inDegrees.set(neighbour, inDegrees.get(neighbour) - 1)
          // if in-degree becomes 0
          if (inDegrees.get(neighbour) === 0) {
            // then put the neighboring node to the queue
            queue.push(neighbour)
          }
        })
      }
      // vertices not same as visited = cycle found
      if (vertices !== visited) {
        errors.add('Can’t save field because it causes a circular reference')
      }
    }
  } else if (parsedTree.type === JSEPNode.BINARY_EXP) {
    if (!availableBinOps.includes(parsedTree.operator)) {
      errors.add(`'${parsedTree.operator}' operation is not available`)
    }
    validateAgainstMeta(parsedTree.left, errors)
    validateAgainstMeta(parsedTree.right, errors)
  } else if (parsedTree.type === JSEPNode.LITERAL || parsedTree.type === JSEPNode.UNARY_EXP) {
    // do nothing
  } else if (parsedTree.type === JSEPNode.COMPOUND) {
    if (parsedTree.body.length) {
      errors.add('Can’t save field because the formula is invalid')
    }
  } else {
    errors.add('Can’t save field because the formula is invalid')
  }
  return errors
}

function validateAgainstType(parsedTree: any, expectedType: string, func: any, typeErrors = new Set()) {
  if (parsedTree === false || typeof parsedTree === 'undefined') {
    return typeErrors
  }
  if (parsedTree.type === JSEPNode.LITERAL) {
    if (typeof func === 'function') {
      func(parsedTree.value)
    } else if (expectedType === formulaTypes.NUMERIC) {
      if (typeof parsedTree.value !== 'number') {
        typeErrors.add('Numeric type is expected')
      }
    } else if (expectedType === formulaTypes.STRING) {
      if (typeof parsedTree.value !== 'string') {
        typeErrors.add('string type is expected')
      }
    }
  } else if (parsedTree.type === JSEPNode.IDENTIFIER) {
    const col = columns.value.find((c) => c.title === parsedTree.name)

    if (col === undefined) {
      return
    }

    if (col.uidt === UITypes.Formula) {
      const foundType = getRootDataType(jsep((col as any).formula_raw))
      if (foundType === 'N/A') {
        typeErrors.add(`Not supported to reference column ${col.title}`)
      } else if (expectedType !== foundType) {
        typeErrors.add(`Type ${expectedType} is expected but found Type ${foundType}`)
      }
    } else {
      switch (col.uidt) {
        // string
        case UITypes.SingleLineText:
        case UITypes.LongText:
        case UITypes.MultiSelect:
        case UITypes.SingleSelect:
        case UITypes.PhoneNumber:
        case UITypes.Email:
        case UITypes.URL:
          if (expectedType !== formulaTypes.STRING) {
            typeErrors.add(
              `Column '${parsedTree.name}' with ${formulaTypes.STRING} type is found but ${expectedType} type is expected`,
            )
          }
          break

        // numeric
        case UITypes.Year:
        case UITypes.Number:
        case UITypes.Decimal:
        case UITypes.Rating:
        case UITypes.Count:
        case UITypes.AutoNumber:
        case UITypes.Currency:
          if (expectedType !== formulaTypes.NUMERIC) {
            typeErrors.add(
              `Column '${parsedTree.name}' with ${formulaTypes.NUMERIC} type is found but ${expectedType} type is expected`,
            )
          }
          break

        // date
        case UITypes.Date:
        case UITypes.DateTime:
        case UITypes.CreateTime:
        case UITypes.LastModifiedTime:
          if (expectedType !== formulaTypes.DATE) {
            typeErrors.add(
              `Column '${parsedTree.name}' with ${formulaTypes.DATE} type is found but ${expectedType} type is expected`,
            )
          }
          break

        // not supported
        case UITypes.ForeignKey:
        case UITypes.Attachment:
        case UITypes.ID:
        case UITypes.Time:
        case UITypes.Percent:
        case UITypes.Duration:
        case UITypes.Rollup:
        case UITypes.Lookup:
        case UITypes.Barcode:
        case UITypes.Button:
        case UITypes.Checkbox:
        case UITypes.Collaborator:
        default:
          typeErrors.add(`Not supported to reference column '${parsedTree.name}'`)
          break
      }
    }
  } else if (parsedTree.type === JSEPNode.UNARY_EXP || parsedTree.type === JSEPNode.BINARY_EXP) {
    if (expectedType !== formulaTypes.NUMERIC) {
      // parsedTree.name won't be available here
      typeErrors.add(`${formulaTypes.NUMERIC} type is found but ${expectedType} type is expected`)
    }
  } else if (parsedTree.type === JSEPNode.CALL_EXP) {
    if (formulas[parsedTree.callee.name]?.type && expectedType !== formulas[parsedTree.callee.name].type) {
      typeErrors.add(`${expectedType} not matched with ${formulas[parsedTree.callee.name].type}`)
    }
  }
  return typeErrors
}

function getRootDataType(parsedTree: any): any {
  // given a parse tree, return the data type of it
  if (parsedTree.type === JSEPNode.CALL_EXP) {
    return formulas[parsedTree.callee.name].type
  } else if (parsedTree.type === JSEPNode.IDENTIFIER) {
    const col = columns.value.find((c) => c.title === parsedTree.name) as Record<string, any>
    if (col?.uidt === UITypes.Formula) {
      return getRootDataType(jsep(col?.formula_raw))
    } else {
      switch (col?.uidt) {
        // string
        case UITypes.SingleLineText:
        case UITypes.LongText:
        case UITypes.MultiSelect:
        case UITypes.SingleSelect:
        case UITypes.PhoneNumber:
        case UITypes.Email:
        case UITypes.URL:
          return formulaTypes.STRING

        // numeric
        case UITypes.Year:
        case UITypes.Number:
        case UITypes.Decimal:
        case UITypes.Rating:
        case UITypes.Count:
        case UITypes.AutoNumber:
          return formulaTypes.NUMERIC

        // date
        case UITypes.Date:
        case UITypes.DateTime:
        case UITypes.CreateTime:
        case UITypes.LastModifiedTime:
          return formulaTypes.DATE

        // not supported
        case UITypes.ForeignKey:
        case UITypes.Attachment:
        case UITypes.ID:
        case UITypes.Time:
        case UITypes.Currency:
        case UITypes.Percent:
        case UITypes.Duration:
        case UITypes.Rollup:
        case UITypes.Lookup:
        case UITypes.Barcode:
        case UITypes.Button:
        case UITypes.Checkbox:
        case UITypes.Collaborator:
        default:
          return 'N/A'
      }
    }
  } else if (parsedTree.type === JSEPNode.BINARY_EXP || parsedTree.type === JSEPNode.UNARY_EXP) {
    return formulaTypes.NUMERIC
  } else if (parsedTree.type === JSEPNode.LITERAL) {
    return typeof parsedTree.value
  } else {
    return 'N/A'
  }
}

function isCurlyBracketBalanced() {
  // count number of opening curly brackets and closing curly brackets
  const cntCurlyBrackets = (formulaRef.value.$el.value.match(/\{|}/g) || []).reduce(
    (acc: Record<number, number>, cur: number) => {
      acc[cur] = (acc[cur] || 0) + 1
      return acc
    },
    {},
  )
  return (cntCurlyBrackets['{'] || 0) === (cntCurlyBrackets['}'] || 0)
}

function appendText(item: Record<string, any>) {
  const text = item.text
  const len = wordToComplete.value?.length || 0

  if (item.type === 'function') {
    vModel.value.formula_raw = insertAtCursor(formulaRef.value.$el, text, len, 1)
  } else if (item.type === 'column') {
    vModel.value.formula_raw = insertAtCursor(formulaRef.value.$el, `{${text}}`, len + +!isCurlyBracketBalanced())
  } else {
    vModel.value.formula_raw = insertAtCursor(formulaRef.value.$el, text, len)
  }
  autocomplete.value = false
  wordToComplete.value = ''
  if (item.type === 'function' || item.type === 'op') {
    // if function / operator is chosen, display columns only
    suggestion.value = suggestionsList.value.filter((f) => f.type === 'column')
  } else {
    // show all options if column is chosen
    suggestion.value = suggestionsList.value
  }
}

const handleInputDeb = useDebounceFn(function () {
  handleInput()
}, 250)

function handleInput() {
  selected.value = 0
  suggestion.value = []
  const query = getWordUntilCaret(formulaRef.value.$el)
  const parts = query.split(/\W+/)
  wordToComplete.value = parts.pop() || ''
  suggestion.value = acTree.value
    .complete(wordToComplete.value)
    ?.sort((x: Record<string, any>, y: Record<string, any>) => sortOrder[x.type] - sortOrder[y.type])
  if (!isCurlyBracketBalanced()) {
    suggestion.value = suggestion.value.filter((v: Record<string, any>) => v.type === 'column')
  }
  autocomplete.value = !!suggestion.value.length
}

function selectText() {
  if (suggestion.value && selected.value > -1 && selected.value < suggestion.value.length) {
    appendText(suggestion.value[selected.value])
  }
}

function suggestionListUp() {
  if (suggestion.value) {
    selected.value = --selected.value > -1 ? selected.value : suggestion.value.length - 1
    scrollToSelectedOption()
  }
}

function suggestionListDown() {
  if (suggestion.value) {
    selected.value = ++selected.value % suggestion.value.length
    scrollToSelectedOption()
  }
}

function scrollToSelectedOption() {
  nextTick(() => {
    if (sugOptionsRef.value[selected.value]) {
      try {
        sugListRef.value.$el.scrollTo({
          top: sugOptionsRef.value[selected.value].$el.offsetTop,
          behavior: 'smooth',
        })
      } catch (e) {}
    }
  })
}

// set default value
vModel.value.formula_raw = (column?.value?.colOptions as Record<string, any>)?.formula_raw || ''

// set additional validations
setAdditionalValidations({
  ...validators,
})

onMounted(() => {
  jsep.plugins.register(jsepCurlyHook)
})
</script>

<template>
  <div class="formula-wrapper">
    <a-form-item v-bind="validateInfos.formula_raw" label="Formula">
      <a-textarea
        ref="formulaRef"
        v-model:value="vModel.formula_raw"
        class="mb-2 nc-formula-input"
        @keydown.down.prevent="suggestionListDown"
        @keydown.up.prevent="suggestionListUp"
        @keydown.enter.prevent="selectText"
        @change="handleInputDeb"
      />
    </a-form-item>
    <div class="text-gray-600 mt-2 mb-4 prose-sm">
      Hint: Use {} to reference columns, e.g: {column_name}. For more, please check out
      <a class="prose-sm" href="https://docs.nocodb.com/setup-and-usages/formulas#available-formula-features" target="_blank">
        Formulas.
      </a>
    </div>

    <div class="h-[250px] overflow-auto scrollbar-thin-primary">
      <a-list ref="sugListRef" :data-source="suggestion" :locale="{ emptyText: 'No suggested formula was found' }">
        <template #renderItem="{ item, index }">
          <a-list-item
            :ref="
              (el) => {
                sugOptionsRef[index] = el
              }
            "
            class="cursor-pointer"
            @click.prevent.stop="appendText(item)"
          >
            <a-list-item-meta>
              <template #title>
                <div class="flex">
                  <a-col :span="6">
                    <span class="prose-sm text-gray-600">{{ item.text }}</span>
                  </a-col>
                  <a-col :span="18">
                    <div v-if="item.type === 'function'" class="text-xs text-gray-500">
                      {{ item.description }} <br /><br />
                      Syntax: <br />
                      {{ item.syntax }} <br /><br />
                      Examples: <br />
                      <div v-for="(example, idx) of item.examples" :key="idx">
                        <div>({{ idx + 1 }}): {{ example }}</div>
                      </div>
                    </div>
                    <div v-if="item.type === 'column'" class="float-right mr-5 -mt-2">
                      <a-badge-ribbon :text="item.uidt" color="gray" />
                    </div>
                  </a-col>
                </div>
              </template>

              <template #avatar>
                <mdi-function v-if="item.type === 'function'" class="text-lg" />
                <mdi-calculator v-if="item.type === 'op'" class="text-lg" />
                <component :is="item.icon" v-if="item.type === 'column'" class="text-lg" />
              </template>
            </a-list-item-meta>
          </a-list-item>
        </template>
      </a-list>
    </div>
  </div>
</template>
