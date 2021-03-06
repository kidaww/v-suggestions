<template>
    <div class="v-suggestions" @mouseenter="handleMouseEnter" @mouseleave="handleMouseLeave">
        <input type="text" :class="extendedOptions.inputClass"
               ref="input"
               v-bind="$attrs"
               v-on:keydown="onKeyDown"
               v-on:blur="hideItems"
               v-on:focus="onFocus"
               v-on:paste="onPaste"
               v-model="query"
               :autocomplete="Math.random()"
               :placeholder="extendedOptions.placeholder">
        <div class="suggestions">
            <ul class="items" v-show="items.length > 0 && showItems === true">
                <li class="item"
                    :key="index"
                    v-for="(item, index) in items"
                    @click.prevent="selectItem(index)"
                    v-bind:class="{ 'is-active': index === activeItemIndex }">
                    <slot name="item"
                          :item="item">
                        {{item}}
                    </slot>
                </li>
            </ul>
        </div>
    </div>
</template>
<script>
    import debounce from 'debounce'
    import './Suggestions.css'

    export default {
      inheritAttributes: true,
      props: {
        options: {
          type: Object,
          default: {}
        },
        onInputChange: {
          type: Function,
          required: true
        },
        onItemSelected: {
          type: Function
        },
        value: {
          type: String,
          required: true
        },
        onFocusExecution: {
          type: Boolean
        },
        mouseMoveDetection: {
          type: Boolean,
        }
      },
      data () {
        const defaultOptions = {
          debounce: 0,
          placeholder: '',
          inputClass: 'input',
          ignoreNull: false
        }
        const extendedOptions = Object.assign({}, defaultOptions, this.options)
        return {
          extendedOptions,
          query: this.value,
          lastSetQuery: null,
          items: [],
          activeItemIndex: -1,
          showItems: false
        }
      },
      beforeMount () {
        if (this.extendedOptions.debounce !== 0) {
          this.onQueryChanged = debounce(this.onQueryChanged, this.extendedOptions.debounce)
        }
      },
      watch: {
        'query': function (newValue, oldValue) {
          if (newValue === this.lastSetQuery) {
            this.lastSetQuery = null
            return
          }
          this.onQueryChanged(newValue)
          this.$emit('input', newValue)
        },
        'value': function (newValue, oldValue) {
          this.setInputQuery(newValue)
        },
        'options.inputClass': function (newValue) {
          this.extendedOptions.inputClass = newValue || 'input'
        }
      },
      methods: {
        handleMouseEnter(event) {
          if (!this.mouseMoveDetection) return;

          this.showItems = true;
        },
        handleMouseLeave(event) {
          if (!this.mouseMoveDetection) return;

          this.showItems = false;
        },
        onPaste(event) {
          this.$emit('paste', event);
        },
        setFocus () {
          this.$refs.input.focus()
        },
        onFocus () {
          this.showItems = true
          if (this.onFocusExecution && !this.query) {
            this.onQueryChanged(this.query)
          }
        },
        onItemSelectedDefault (item) {
          if (typeof item === 'string') {
            this.$emit('input', item)
            this.setInputQuery(item)
            this.showItems = false
                    // console.log('change value')
          }
        },
        hideItems () {
          setTimeout(() => {
            if (this.activeItemIndex === 0 && this.extendedOptions.ignoreNull) {
              return
            }
            this.showItems = false
          }, 300)
        },
        showResults () {
          this.showItems = true
        },
        setInputQuery (value) {
          this.lastSetQuery = value
          this.query = value
        },
        onKeyDown (e) {
          switch (e.keyCode) {
            case 40:
              this.highlightItem('down')
              e.preventDefault()
              break
            case 38:
              this.highlightItem('up')
              e.preventDefault()
              break
            case 13:
              this.selectItem()
              e.preventDefault()
              break
            case 27:
              this.showItems = false
              e.preventDefault()
              break
            default:
              return true
          }
        },
        selectItem (index) {
          if (index === 0 && this.extendedOptions.ignoreNull) {
            this.activeItemIndex = 0
            return
          }
          let item = null
          if (typeof index === 'undefined') {
            if (this.activeItemIndex === -1) {
              return
            }
            item = this.items[this.activeItemIndex]
          } else {
            item = this.items[index]
          }
          if (!item) {
            return
          }
          if (this.onItemSelected) {
            this.onItemSelected(item)
          } else {
            this.onItemSelectedDefault(item)
          }
          this.hideItems()
        },
        highlightItem (direction) {
          if (this.items.length === 0) {
            return
          }
          let selectedIndex = this.items.findIndex((item, index) => {
            return index === this.activeItemIndex
          })
          if (selectedIndex === -1) {
                    // nothing selected
            if (direction === 'down') {
              selectedIndex = 0
            } else {
              selectedIndex = this.items.length - 1
            }
          } else {
            this.activeIndexItem = 0
            if (direction === 'down') {
              selectedIndex++
              if (selectedIndex === this.items.length) {
                selectedIndex = 0
              }
            } else {
              selectedIndex--
              if (selectedIndex < 0) {
                selectedIndex = this.items.length - 1
              }
            }
          }
          this.activeItemIndex = selectedIndex
        },
        setItems (items) {
          this.items = items
          this.activeItemIndex = -1
          this.showItems = true
        },
        onQueryChanged (value) {
          const result = this.onInputChange(value)
          this.items = []
          if (typeof result === 'undefined' || typeof result === 'boolean' || result === null) {
            return
          }
          if (result instanceof Array) {
            this.setItems(result)
          } else if (typeof result.then === 'function') {
            result.then(items => {
              this.setItems(items)
            })
          }
        }
      }
    }
</script>
