<template>
    <li>
        <a class="item dropdown-item"
            :class="{ 'is-bold has-background-warning-light': item.selected }"
            @click="select">
            <div class="level">
                <div class="level-left">
                    <div class="level-item is-marginless"
                        v-if="hasChildren">
                        <span class="icon is-small"
                            @click.stop="toggle">
                            <fa icon="minus-square"
                                v-if="isExpanded"/>
                            <fa icon="plus-square"
                                v-else/>
                        </span>
                    </div>
                    <div class="level-item pl-3">
                        <slot name="item"
                            :item="item"
                            :highlight="highlight">
                            <!-- eslint-disable-next-line vue/no-v-html -->
                            <span v-html="highlight(item.name)"/>
                            <span class="icon has-text-info"
                                v-if="item.description"
                                v-tooltip="i18n(item.description)">
                            <fa icon="question-circle"/>
                        </span>
                        </slot>
                    </div>
                </div>
                <div class="level-right"
                    v-if="item.selected">
                    <div class="level-item">
                        <template v-if="state.editable">
                            <slot name="controls"
                                :item="item"/>
                            <a class="button is-naked is-small"
                                @click.stop="edit">
                                <span class="icon">
                                    <fa icon="pencil-alt"/>
                                </span>
                            </a>
                            <confirmation @confirm="destroy">
                                <a class="is-naked mr-2">
                                    <span class="icon">
                                        <fa icon="trash-alt"/>
                                    </span>
                                </a>
                            </confirmation>
                        </template>
                        <a class="delete is-small"
                            @click.stop="clear"/>
                    </div>
                </div>
            </div>
        </a>
        <items :items="item.items"
            :parent-id="item.id"
            @selected="$emit('selected', $event)"
            @deselected="$emit('deselected', $event)"
            @moved="$emit('moved', $event)"
            @update:model-value="$emit('update:modelValue', $event)"
            v-show="!hasChildren || isExpanded"
            v-if="canHaveChildren">
            <template #item="props">
                <slot name="item"
                    v-bind="props"/>
            </template>
            <template #controls="props">
                <slot name="controls"
                    v-bind="props"/>
            </template>
        </items>
    </li>
</template>

<script>
import { defineAsyncComponent } from 'vue';
import { FontAwesomeIcon as Fa } from '@fortawesome/vue-fontawesome';
import { library } from '@fortawesome/fontawesome-svg-core';
import {
    faMinusSquare, faPlusSquare, faPencilAlt, faTrashAlt, faQuestionCircle,
} from '@fortawesome/free-solid-svg-icons';
import Confirmation from '@liberu-ui/confirmation/bulma';

library.add(faMinusSquare, faPlusSquare, faPencilAlt, faTrashAlt, faQuestionCircle);

export default {
    name: 'Item',

    components: { Confirmation, Fa, Items: defineAsyncComponent(() => import('./Items.vue')) },

    inject: [
        'errorHandler', 'route', 'state', 'http', 'i18n', 'is', 'routePrefix',
    ],

    props: {
        item: {
            type: Object,
            required: true,
        },
        splice: {
            type: Function,
            required: true,
        },
    },

    emits: ['deselected', 'moved', 'selected', 'update:modelValue'],

    computed: {
        canHaveChildren() {
            return !!this.item.items;
        },
        hasChildren() {
            return this.item.items?.length > 0;
        },
        isExpanded() {
            return this.state.expanded
                .some(current => this.is(current, this.item));
        },
    },

    mounted() {
        this.$el.__vue__ = this;
    },

    methods: {
        bold(label, arg) {
            let from;

            try {
                from = new RegExp(`(${arg})`, 'gi');
            } catch {
                from = arg;
            }

            return `${label}`.replace(from, '<b>$1</b>');
        },
        clear() {
            this.item.selected = false;
            this.$emit('update:modelValue', null);
            this.$emit('deselected', this.item);
            this.state.selected = null;
        },
        destroy() {
            const routePrefix = this.routePrefix(this.item);
            const route = this.route(`${routePrefix}.destroy`, this.item.id);

            this.http.delete(route).then(() => {
                this.splice(this.item.id);
                this.state.selected = null;
            }).catch(this.errorHandler);
        },
        edit() {
            this.state.item = this.state.selected;
            this.state.original = this.state.selected.name;
        },
        highlight(label) {
            return this.state.query.toLowerCase().split(' ')
                .filter(arg => arg !== '')
                .reduce((label, arg) => this.bold(label, arg), label);
        },
        select() {
            if (this.state.selected) {
                this.state.selected.selected = false;
            }

            this.item.selected = true;
            this.state.selected = this.item;
            const input = this.state.objects ? this.item : this.item.id;
            this.$emit('update:modelValue', input);
            this.$emit('selected', this.item);
        },
        toggle() {
            if (!this.hasChildren) {
                return;
            }

            const index = this.state.expanded
                .findIndex(current => this.is(current, this.item));

            if (index >= 0) {
                this.state.expanded.splice(index, 1);
            } else {
                this.state.expanded.push(this.item);
            }
        },
    },
};
</script>

<style lang="scss">
    a.item.dropdown-item {
        padding: 0.5em 0.7em;
    }
</style>
