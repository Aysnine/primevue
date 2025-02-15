<template>
    <div ref="container" :class="containerClass" @click="onClick">
        <div class="p-hidden-accessible">
            <input ref="focusInput" type="text" role="listbox" :id="inputId" readonly :disabled="disabled" @focus="onFocus" @blur="onBlur" @keydown="onKeyDown" :tabindex="tabindex"
                aria-haspopup="true" :aria-expanded="overlayVisible" :aria-labelledby="ariaLabelledBy"/>
        </div>
        <div class="p-multiselect-label-container">
            <div :class="labelClass">
                <slot name="value" :value="modelValue" :placeholder="placeholder">
                    <template v-if="display === 'comma'">
                        {{label || 'empty'}}
                    </template>
                    <template v-else-if="display === 'chip'">
                        <div v-for="item of modelValue" class="p-multiselect-token" :key="getLabelByValue(item)">
                            <span class="p-multiselect-token-label">{{getLabelByValue(item)}}</span>
                            <span v-if="!disabled" class="p-multiselect-token-icon pi pi-times-circle" @click="removeChip(item)"></span>
                        </div>
                        <template v-if="!modelValue || modelValue.length === 0">{{placeholder || 'empty'}}</template>
                    </template>
                </slot>
            </div>
        </div>
        <div class="p-multiselect-trigger">
            <span class="p-multiselect-trigger-icon pi pi-chevron-down"></span>
        </div>
        <Teleport :to="appendTarget" :disabled="appendDisabled">
            <transition name="p-connected-overlay" @enter="onOverlayEnter" @leave="onOverlayLeave" @after-leave="onOverlayAfterLeave">
                <div :ref="overlayRef" :class="panelStyleClass" v-if="overlayVisible" @click="onOverlayClick">
                    <slot name="header" :value="modelValue" :options="visibleOptions"></slot>
                    <div class="p-multiselect-header">
                        <div class="p-checkbox p-component" @click="onToggleAll" role="checkbox" :aria-checked="allSelected">
                            <div class="p-hidden-accessible">
                                <input type="checkbox" readonly @focus="onHeaderCheckboxFocus" @blur="onHeaderCheckboxBlur">
                            </div>
                            <div :class="['p-checkbox-box', {'p-highlight': allSelected, 'p-focus': headerCheckboxFocused}]" role="checkbox" :aria-checked="allSelected">
                                <span :class="['p-checkbox-icon', {'pi pi-check': allSelected}]"></span>
                            </div>
                        </div>
                        <div v-if="filter" class="p-multiselect-filter-container">
                            <input type="text" ref="filterInput" v-model="filterValue" class="p-multiselect-filter p-inputtext p-component" :placeholder="filterPlaceholder" @input="onFilterChange">
                            <span class="p-multiselect-filter-icon pi pi-search"></span>
                        </div>
                        <button class="p-multiselect-close p-link" @click="onCloseClick" type="button" v-ripple>
                            <span class="p-multiselect-close-icon pi pi-times" />
                        </button>
                    </div>
                    <div class="p-multiselect-items-wrapper" :style="{'max-height': scrollHeight}">
                        <ul class="p-multiselect-items p-component" role="listbox" aria-multiselectable="true">
                            <template v-if="!optionGroupLabel">
                                <li v-for="(option, i) of visibleOptions" :class="['p-multiselect-item', {'p-highlight': isSelected(option), 'p-disabled': isOptionDisabled(option)}]" role="option" :aria-selected="isSelected(option)"
                                    :key="getOptionRenderKey(option)" @click="onOptionSelect($event, option)" @keydown="onOptionKeyDown($event, option)" :tabindex="tabindex||'0'" :aria-label="getOptionLabel(option)"  v-ripple>
                                    <div class="p-checkbox p-component">
                                        <div :class="['p-checkbox-box', {'p-highlight': isSelected(option)}]">
                                            <span :class="['p-checkbox-icon', {'pi pi-check': isSelected(option)}]"></span>
                                        </div>
                                    </div>
                                    <slot name="option" :option="option" :index="i">
                                        <span>{{getOptionLabel(option)}}</span>
                                    </slot>
                                </li>
                            </template>
                            <template v-else>
                                <template v-for="(optionGroup, i) of visibleOptions" :key="getOptionGroupRenderKey(optionGroup)">
                                    <li  class="p-multiselect-item-group">
                                        <slot name="optiongroup" :option="optionGroup" :index="i">{{getOptionGroupLabel(optionGroup)}}</slot>
                                    </li>
                                    <li v-for="(option, i) of getOptionGroupChildren(optionGroup)" :class="['p-multiselect-item', {'p-highlight': isSelected(option), 'p-disabled': isOptionDisabled(option)}]" role="option" :aria-selected="isSelected(option)"
                                        :key="getOptionRenderKey(option)" @click="onOptionSelect($event, option)" @keydown="onOptionKeyDown($event, option)" :tabindex="tabindex||'0'" :aria-label="getOptionLabel(option)"  v-ripple>
                                        <div class="p-checkbox p-component">
                                            <div :class="['p-checkbox-box', {'p-highlight': isSelected(option)}]">
                                                <span :class="['p-checkbox-icon', {'pi pi-check': isSelected(option)}]"></span>
                                            </div>
                                        </div>
                                        <slot name="option" :option="option" :index="i">
                                            <span>{{getOptionLabel(option)}}</span>
                                        </slot>
                                    </li>
                                </template>
                            </template>
                            <li v-if="filterValue && (!visibleOptions || (visibleOptions && visibleOptions.length === 0))" class="p-multiselect-empty-message">
                                <slot name="emptyfilter">{{emptyFilterMessageText}}</slot>
                            </li>
                            <li v-else-if="(!options || (options && options.length === 0))" class="p-multiselect-empty-message">
                                <slot name="empty">{{emptyMessageText}}</slot>
                            </li>
                        </ul>
                    </div>
                    <slot name="footer" :value="modelValue" :options="visibleOptions"></slot>
                </div>
            </transition>
        </Teleport>
    </div>
</template>

<script>
import {ConnectedOverlayScrollHandler,ObjectUtils,DomHandler,ZIndexUtils} from 'primevue/utils';
import OverlayEventBus from 'primevue/overlayeventbus';
import {FilterService} from 'primevue/api';
import Ripple from 'primevue/ripple';

export default {
    emits: ['update:modelValue', 'before-show', 'before-hide', 'change', 'show', 'hide', 'filter'],
    props: {
        modelValue: null,
        options: Array,
        optionLabel: null,
        optionValue: null,
        optionDisabled: null,
        optionGroupLabel: null,
        optionGroupChildren: null,
		scrollHeight: {
			type: String,
			default: '200px'
		},
		placeholder: String,
		disabled: Boolean,
        tabindex: String,
        inputId: String,
        dataKey: null,
        filter: Boolean,
        filterPlaceholder: String,
        filterLocale: String,
        filterMatchMode: {
            type: String,
            default: 'contains'
        },
        filterFields: {
            type: Array,
            default: null
        },
        ariaLabelledBy: null,
        appendTo: {
            type: String,
            default: 'body'
        },
        emptyFilterMessage: {
            type: String,
            default: null
        },
        emptyMessage: {
            type: String,
            default: null
        },
        display: {
            type: String,
            default: 'comma'
        },
        panelClass: null
    },
    data() {
        return {
            focused: false,
            headerCheckboxFocused: false,
            filterValue: null,
            overlayVisible: false
        };
    },
    outsideClickListener: null,
    resizeListener: null,
    scrollHandler: null,
    overlay: null,
    beforeUnmount() {
        this.unbindOutsideClickListener();
        this.unbindResizeListener();

        if (this.scrollHandler) {
            this.scrollHandler.destroy();
            this.scrollHandler = null;
        }
        
        if (this.overlay) {
            ZIndexUtils.clear(this.overlay);
            this.overlay = null;
        }
    },
    methods: {
        getOptionLabel(option) {
            return this.optionLabel ? ObjectUtils.resolveFieldData(option, this.optionLabel) : option;
        },
        getOptionValue(option) {
            return this.optionValue ? ObjectUtils.resolveFieldData(option, this.optionValue) : option;
        },
        getOptionRenderKey(option) {
            return this.dataKey ? ObjectUtils.resolveFieldData(option, this.dataKey) : this.getOptionLabel(option);
        },
        getOptionGroupRenderKey(optionGroup) {
            return ObjectUtils.resolveFieldData(optionGroup, this.optionGroupLabel);
        },
        getOptionGroupLabel(optionGroup) {
            return ObjectUtils.resolveFieldData(optionGroup, this.optionGroupLabel);
        },
        getOptionGroupChildren(optionGroup) {
            return ObjectUtils.resolveFieldData(optionGroup, this.optionGroupChildren);
        },
        isOptionDisabled(option) {
            return this.optionDisabled ? ObjectUtils.resolveFieldData(option, this.optionDisabled) : false;
        },
        isSelected(option) {
            let selected = false;
            let optionValue = this.getOptionValue(option);

            if (this.modelValue) {
                for (let val of this.modelValue) {
                    if (ObjectUtils.equals(val, optionValue, this.equalityKey)) {
                        selected = true;
                        break;
                    }
                }
            }

            return selected;
        },
        show() {
            this.$emit('before-show');
            this.overlayVisible = true;
        },
        hide() {
            this.$emit('before-hide');
            this.overlayVisible = false;
        },
        onFocus() {
            this.focused = true;
        },
        onBlur() {
            this.focused = false;
        },
        onHeaderCheckboxFocus() {
            this.headerCheckboxFocused = true;
        },
        onHeaderCheckboxBlur() {
            this.headerCheckboxFocused = false;
        },
        onClick(event) {
            if (!this.disabled && (!this.overlay || !this.overlay.contains(event.target)) && !DomHandler.hasClass(event.target, 'p-multiselect-close')) {
                DomHandler.hasClass(event.target, 'p-multiselect-close');
                if (this.overlayVisible)
                    this.hide();
                else
                    this.show();

                this.$refs.focusInput.focus();
            }
        },
        onCloseClick() {
            this.hide();
        },
        onKeyDown(event) {
            switch(event.which) {
                //down
                case 40:
                    if (this.visibleOptions && !this.overlayVisible && event.altKey) {
                        this.show();
                    }
                break;

                //space
                case 32:
                    if (!this.overlayVisible) {
                        this.show();
                        event.preventDefault();
                    }
                break;

                //enter and escape
                case 13:
                case 27:
                    if (this.overlayVisible) {
                        this.hide();
                        event.preventDefault();
                    }
                break;

                //tab
                case 9:
                    this.hide();
                break;

                default:
                break;
            }
        },
        onOptionSelect(event, option) {
            if (this.disabled || this.isOptionDisabled(option)) {
                return;
            }

            let selected = this.isSelected(option);
            let value = null;

            if (selected)
                value = this.modelValue.filter(val => !ObjectUtils.equals(val, this.getOptionValue(option), this.equalityKey));
            else
                value = [...this.modelValue || [], this.getOptionValue(option)];

            this.$emit('update:modelValue', value);
            this.$emit('change', {originalEvent: event, value: value});
        },
        onOptionKeyDown(event, option) {
            let listItem = event.target;

            switch(event.which) {
                //down
                case 40:
                    var nextItem = this.findNextItem(listItem);
                    if (nextItem) {
                        nextItem.focus();
                    }

                    event.preventDefault();
                break;

                //up
                case 38:
                    var prevItem = this.findPrevItem(listItem);
                    if (prevItem) {
                        prevItem.focus();
                    }

                    event.preventDefault();
                break;

                //enter
                case 13:
                    this.onOptionSelect(event, option);
                    event.preventDefault();
                break;

                default:
                break;
            }
        },
        findNextItem(item) {
            let nextItem = item.nextElementSibling;

            if (nextItem)
                return DomHandler.hasClass(nextItem, 'p-disabled') || DomHandler.hasClass(nextItem, 'p-multiselect-item-group') ? this.findNextItem(nextItem) : nextItem;
            else
                return null;
        },
        findPrevItem(item) {
            let prevItem = item.previousElementSibling;

            if (prevItem)
                return DomHandler.hasClass(prevItem, 'p-disabled') || DomHandler.hasClass(prevItem, 'p-multiselect-item-group') ? this.findPrevItem(prevItem) : prevItem;
            else
                return null;
        },
        onOverlayEnter(el) {
            ZIndexUtils.set('overlay', el, this.$primevue.config.zIndex.overlay);
            this.alignOverlay();
            this.bindOutsideClickListener();
            this.bindScrollListener();
            this.bindResizeListener();

            if (this.filter) {
                this.$refs.filterInput.focus();
            }

            this.$emit('show');
        },
        onOverlayLeave() {
            this.unbindOutsideClickListener();
            this.unbindScrollListener();
            this.unbindResizeListener();
            this.$emit('hide');
            this.overlay = null;
        },
        onOverlayAfterLeave(el) {
            ZIndexUtils.clear(el);
        },
        alignOverlay() {
            if (this.appendDisabled) {
                DomHandler.relativePosition(this.overlay, this.$el);
            }
            else {
                this.overlay.style.minWidth = DomHandler.getOuterWidth(this.$el) + 'px';
                DomHandler.absolutePosition(this.overlay, this.$el);
            }
        },
        bindOutsideClickListener() {
            if (!this.outsideClickListener) {
                this.outsideClickListener = (event) => {
                    if (this.overlayVisible && this.isOutsideClicked(event)) {
                        this.hide();
                    }
                };
                document.addEventListener('click', this.outsideClickListener);
            }
        },
        unbindOutsideClickListener() {
            if (this.outsideClickListener) {
                document.removeEventListener('click', this.outsideClickListener);
                this.outsideClickListener = null;
            }
        },
        bindScrollListener() {
            if (!this.scrollHandler) {
                this.scrollHandler = new ConnectedOverlayScrollHandler(this.$refs.container, () => {
                    if (this.overlayVisible) {
                        this.hide();
                    }
                });
            }

            this.scrollHandler.bindScrollListener();
        },
        unbindScrollListener() {
            if (this.scrollHandler) {
                this.scrollHandler.unbindScrollListener();
            }
        },
        bindResizeListener() {
            if (!this.resizeListener) {
                this.resizeListener = () => {
                    if (this.overlayVisible && !DomHandler.isAndroid()) {
                        this.hide();
                    }
                };
                window.addEventListener('resize', this.resizeListener);
            }
        },
        unbindResizeListener() {
            if (this.resizeListener) {
                window.removeEventListener('resize', this.resizeListener);
                this.resizeListener = null;
            }
        },
        isOutsideClicked(event) {
            return !(this.$el.isSameNode(event.target) || this.$el.contains(event.target) || (this.overlay && this.overlay.contains(event.target)));
        },
        getLabelByValue(val) {
            let option;
            if (this.options) {
                if (this.optionGroupLabel) {
                    for (let optionGroup of this.options) {
                        option = this.findOptionByValue(val, this.getOptionGroupChildren(optionGroup));
                        if (option) {
                            break;
                        }
                    }
                }
                else {
                    option = this.findOptionByValue(val, this.options);
                }
            }

            return option ? this.getOptionLabel(option): null;
        },
        findOptionByValue(val, list) {
            for (let option of list) {
                let optionValue = this.getOptionValue(option);

                if(ObjectUtils.equals(optionValue, val, this.equalityKey)) {
                    return option;
                }
            }

            return null;
        },
        onToggleAll(event) {
            let value = null;

            if (this.allSelected) {
                value = [];
            }
            else if (this.visibleOptions) {
                if (this.optionGroupLabel) {
                    value = [];
                    this.visibleOptions.forEach(optionGroup => value = [...value, ...this.getOptionGroupChildren(optionGroup)]);
                }   
                else  {
                    value = this.visibleOptions.map(option => this.getOptionValue(option));
                }
            }

            this.$emit('update:modelValue', value);
            this.$emit('change', {originalEvent: event, value: value});
        },
        onFilterChange(event) {
            this.$emit('filter', {originalEvent: event, value: event.target.value});
            if (this.overlayVisible) {
                this.alignOverlay();
            }
        },
        overlayRef(el) {
            this.overlay = el;
        },
        removeChip(item) {
            let value = this.modelValue.filter(val => !ObjectUtils.equals(val, item, this.equalityKey));

            this.$emit('update:modelValue', value);
            this.$emit('change', {originalEvent: event, value: value});
        },
        onOverlayClick(event) {
            OverlayEventBus.emit('overlay-click', {
                originalEvent: event,
                target: this.$el
            });
        }
    },
    computed: {
         visibleOptions() {
            if (this.filterValue) {
                if (this.optionGroupLabel) {
                    let filteredGroups = [];
                    for (let optgroup of this.options) {
                        let filteredSubOptions = FilterService.filter(this.getOptionGroupChildren(optgroup), this.searchFields, this.filterValue, this.filterMatchMode, this.filterLocale);
                        if (filteredSubOptions && filteredSubOptions.length) {
                            filteredGroups.push({...optgroup, ...{items: filteredSubOptions}});
                        }
                    }
                    return filteredGroups
                }
                else {
                    return FilterService.filter(this.options, this.searchFields, this.filterValue, 'contains', this.filterLocale);
                }
            }
            else {
                return this.options;
            }
        },
        containerClass() {
            return [
                'p-multiselect p-component p-inputwrapper',
                {
                    'p-multiselect-chip': this.display === 'chip',
                    'p-disabled': this.disabled,
                    'p-focus': this.focused,
                    'p-inputwrapper-filled': this.modelValue && this.modelValue.length,
                    'p-inputwrapper-focus': this.focused || this.overlayVisible
                }
            ];
        },
        labelClass() {
            return [
                'p-multiselect-label',
                {
                    'p-placeholder': this.label === this.placeholder,
                    'p-multiselect-label-empty': !this.placeholder && (!this.modelValue || this.modelValue.length === 0)
                }
            ];
        },
        panelStyleClass() {
            return ['p-multiselect-panel p-component', this.panelClass];
        },
        label() {
            let label;

            if (this.modelValue && this.modelValue.length) {
                label = '';
                for(let i = 0; i < this.modelValue.length; i++) {
                    if(i !== 0) {
                        label += ', ';
                    }

                    label += this.getLabelByValue(this.modelValue[i]);
                }
            }
            else {
                label = this.placeholder;
            }

            return label;
        },
        allSelected() {
            if (this.filterValue && this.filterValue.trim().length > 0) {
                if (this.visibleOptions.length === 0) {
                    return false;
                }

				if (this.optionGroupLabel) {
                    for (let optionGroup of this.visibleOptions) {
                        for (let option of this.getOptionGroupChildren(optionGroup)) {
                            if (!this.isSelected(option)) {
                                return false;
                            }
                        }
                    }
                }
                else {
                    for (let option of this.visibleOptions) {
                        if (!this.isSelected(option)) {
                            return false;
                        }
                    }
                }

                return true;
            }
            else {
                if (this.modelValue && this.options) {
                    let optionCount = 0;
                    if (this.optionGroupLabel)
                        this.options.forEach(optionGroup => optionCount += this.getOptionGroupChildren(optionGroup).length);
                    else
                        optionCount = this.options.length;

                    return optionCount > 0 && optionCount === this.modelValue.length;
                }
                
                return false;
            }
        },
        equalityKey() {
            return this.optionValue ? null : this.dataKey;
        },
        searchFields() {
            return this.filterFields || [this.optionLabel];
        },
        emptyFilterMessageText() {
            return this.emptyFilterMessage || this.$primevue.config.locale.emptyFilterMessage;
        },
        emptyMessageText() {
            return this.emptyMessage || this.$primevue.config.locale.emptyMessage;
        },
        appendDisabled() {
            return this.appendTo === 'self';
        },
        appendTarget() {
            return this.appendDisabled ? null : this.appendTo;
        }
    },
    directives: {
        'ripple': Ripple
    }
}
</script>

<style>
.p-multiselect {
    display: inline-flex;
    cursor: pointer;
    position: relative;
    user-select: none;
}

.p-multiselect-trigger {
    display: flex;
    align-items: center;
    justify-content: center;
    flex-shrink: 0;
}

.p-multiselect-label-container {
    overflow: hidden;
    flex: 1 1 auto;
    cursor: pointer;
}

.p-multiselect-label  {
    display: block;
    white-space: nowrap;
    cursor: pointer;
    overflow: hidden;
    text-overflow: ellipsis;
}

.p-multiselect-label-empty {
    overflow: hidden;
    visibility: hidden;
}

.p-multiselect-token {
    cursor: default;
    display: inline-flex;
    align-items: center;
    flex: 0 0 auto;
}

.p-multiselect-token-icon {
    cursor: pointer;
}

.p-multiselect .p-multiselect-panel {
    min-width: 100%;
}

.p-multiselect-panel {
    position: absolute;
}

.p-multiselect-items-wrapper {
    overflow: auto;
}

.p-multiselect-items {
    margin: 0;
    padding: 0;
    list-style-type: none;
}

.p-multiselect-item {
    cursor: pointer;
    display: flex;
    align-items: center;
    font-weight: normal;
    white-space: nowrap;
    position: relative;
    overflow: hidden;
}

.p-multiselect-item-group {
    cursor: auto;
}

.p-multiselect-header {
    display: flex;
    align-items: center;
    justify-content: space-between;
}

.p-multiselect-filter-container {
    position: relative;
    flex: 1 1 auto;
}

.p-multiselect-filter-icon {
    position: absolute;
    top: 50%;
    margin-top: -.5rem;
}

.p-multiselect-filter-container .p-inputtext {
    width: 100%;
}

.p-multiselect-close {
    display: flex;
    align-items: center;
    justify-content: center;
    flex-shrink: 0;
    overflow: hidden;
    position: relative;
}

.p-fluid .p-multiselect {
    display: flex;
}
</style>
