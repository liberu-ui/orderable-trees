<template>
    <draggable class="drag-area"
        :list="items"
        :animation="150"
        :group="{ name: 'items', put: unique }"
        :move="checkMove"
        :item-key="item => `${item.group}-${item.id}`"
        handle=".dropdown-item"
        :disabled="state.query !== '' || !!state.item"
        @change="change"
        @end="endDragging"
        @start="startDragging"
        tag="ul">
        <template #item="{ element }">
            <item :item="element"
                :splice="splice"
                @moved="$emit('moved', $event)"
                @update:modelValue="$emit('update:modelValue', $event)">
                <template #item="props">
                    <slot name="item"
                        v-bind="props"/>
                </template>
                <template #controls="props">
                    <slot name="controls"
                        v-bind="props"/>
                </template>
            </item>
        </template>
    </draggable>
</template>

<script>
import Draggable from 'vuedraggable';
import Item from './Item.vue';

export default {
    name: 'Items',

    components: { Draggable, Item },

    inject: ['state'],

    props: {
        items: {
            type: Array,
            required: true,
        },
        parentId: {
            type: Number,
            default: null,
        },
    },

    emits: ['moved', 'update:modelValue'],

    mounted() {
        this.$el.__vue__ = this;
    },

    methods: {
        checkMove({ draggedContext: { element }, relatedContext }) {
            return !relatedContext.element
                || relatedContext.element.group === element.group;
        },
        change(event) {
            if (event.moved) {
                this.move(event.moved);
            } else if (event.added) {
                this.move(event.added);
            }
        },
        endDragging() {
            this.state.dragging = null;
        },
        splice(item) {
            const index = this.items.findIndex(({ id }) => id === item.id);

            this.items.splice(index, 1);
        },
        move({ element, newIndex }) {
            newIndex++;
            const payload = { ...element, parentId: this.parentId, newIndex };
            this.$emit('moved', payload);
        },
        startDragging(event) {
            if (this.state.selected) {
                this.state.selected.selected = false;
                this.state.selected = null;
            }

            // eslint-disable-next-line no-underscore-dangle
            this.state.dragging = event.item.__vue__.$options.propsData.item;
        },
        unique(to) {
            // eslint-disable-next-line no-underscore-dangle
            const instanceProps = to.el.__vue__.$options.propsData;
            const items = instanceProps.list || instanceProps.items;

            return !items.some(item => item.name === this.state.dragging.name);
        },
    },
};
</script>
