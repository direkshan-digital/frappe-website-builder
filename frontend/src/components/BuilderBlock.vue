<template>
	<component
		:is="getComponentName(block)"
		:selected="isSelected"
		@click="handleClick"
		@dblclick="handleDoubleClick"
		@contextmenu="triggerContextMenu($event)"
		@mouseover="handleMouseOver"
		@mouseleave="handleMouseLeave"
		:data-block-id="block.blockId"
		:draggable="draggable"
		:class="classes"
		v-bind="attributes"
		:style="styles"
		v-if="showBlock"
		ref="component">
		<BuilderBlock
			:data="data"
			:block="child"
			:breakpoint="breakpoint"
			:preview="preview"
			:isChildOfComponent="block.isExtendedFromComponent() || isChildOfComponent"
			:key="child.blockId"
			v-for="child in block.getChildren()" />
	</component>
	<teleport to="#overlay" v-if="canvasProps?.overlayElement && !preview && Boolean(canvasProps)">
		<!-- prettier-ignore -->
		<BlockEditor
			ref="editor"
			v-show="!isEditable"
			v-if="loadEditor"
			:block="block"
			:breakpoint="breakpoint"
			:editable="isEditable"
			:isSelected="isSelected"
			:target="(target as HTMLElement)" />
	</teleport>
</template>
<script setup lang="ts">
import Block from "@/utils/block";
import { setFont } from "@/utils/fontManager";
import { computed, inject, nextTick, onMounted, reactive, ref, useAttrs, watch, watchEffect } from "vue";

import getBlockTemplate from "@/utils/blockTemplate";
import { getDataForKey } from "@/utils/helpers";
import { useDraggableBlock } from "@/utils/useDraggableBlock";
import useStore from "../store";
import BlockEditor from "./BlockEditor.vue";
import BlockHTML from "./BlockHTML.vue";
import DataLoaderBlock from "./DataLoaderBlock.vue";
import TextBlock from "./TextBlock.vue";

const component = ref<HTMLElement | InstanceType<typeof TextBlock> | null>(null);
const attrs = useAttrs();
const store = useStore();
const editor = ref<InstanceType<typeof BlockEditor> | null>(null);

const props = defineProps({
	block: {
		type: Block,
		required: true,
	},
	isChildOfComponent: {
		type: Boolean,
		default: false,
	},
	breakpoint: {
		type: String,
		default: "desktop",
	},
	preview: {
		type: Boolean,
		default: false,
	},
	data: {
		type: Object,
		default: null,
	},
});

defineOptions({
	inheritAttrs: false,
});

const draggable = computed(() => {
	// TODO: enable this
	return !props.block.isRoot() && !props.preview && false;
});

const isHovered = ref(false);
const isSelected = ref(false);

const getComponentName = (block: Block) => {
	if (block.isRepeater()) {
		return DataLoaderBlock;
	}
	if (block.isText() || block.isLink() || block.isButton()) {
		return TextBlock;
	} else if (block.isHTML()) {
		return BlockHTML;
	} else {
		return block.getTag();
	}
};

const classes = computed(() => {
	return [attrs.class, "__builder_component__", "outline-none", "select-none", ...props.block.getClasses()];
});

const attributes = computed(() => {
	const attribs = { ...props.block.getAttributes(), ...attrs } as { [key: string]: any };
	if (
		props.block.isText() ||
		props.block.isHTML() ||
		props.block.isLink() ||
		props.block.isButton() ||
		props.block.isRepeater()
	) {
		attribs.block = props.block;
		attribs.preview = props.preview;
		attribs.breakpoint = props.breakpoint;
		attribs.data = props.data;
	}

	if (props.data) {
		if (props.block.getDataKey("type") === "attribute") {
			attribs[props.block.getDataKey("property") as string] =
				getDataForKey(props.data, props.block.getDataKey("key")) ||
				attribs[props.block.getDataKey("property") as string];
		}
	}

	if (props.block.isInput()) {
		attribs.readonly = true;
	}
	return attribs;
});

const canvasProps = !props.preview ? (inject("canvasProps") as CanvasProps) : null;

const target = computed(() => {
	if (!component.value) return null;
	if (component.value instanceof HTMLElement || component.value instanceof SVGElement) {
		return component.value;
	} else {
		return component.value.component;
	}
});

const styles = computed(() => {
	let dynamicStyles = {};
	if (props.data) {
		if (props.block.getDataKey("type") === "style") {
			dynamicStyles = {
				[props.block.getDataKey("property") as string]: getDataForKey(
					props.data,
					props.block.getDataKey("key"),
				),
			};
		}
	}

	const styleMap = {
		...props.block.getStyles(props.breakpoint),
		...props.block.getEditorStyles(),
		...dynamicStyles,
	} as BlockStyleMap;
	// escape space in font family
	if (styleMap.fontFamily) {
		styleMap.fontFamily = (styleMap.fontFamily as string).replace(/ /g, "\\ ");
	}
	return styleMap;
});

const loadEditor = computed(() => {
	return (
		target.value &&
		props.block.getStyle("display") !== "none" &&
		((isSelected.value && props.breakpoint === store.activeBreakpoint) ||
			(isHovered.value && store.hoveredBreakpoint === props.breakpoint)) &&
		!canvasProps?.scaling &&
		!canvasProps?.panning
	);
});

const emit = defineEmits(["mounted"]);

watchEffect(() => {
	setFont(props.block.getStyle("fontFamily") as string, props.block.getStyle("fontWeight") as string);
});

onMounted(async () => {
	await nextTick();
	emit("mounted", target.value);

	if (draggable.value) {
		useDraggableBlock(
			props.block,
			component.value as HTMLElement,
			reactive({ ghostScale: canvasProps?.scale || 1 }),
		);
	}
});

const isEditable = computed(() => {
	// to ensure it is right block and not on different breakpoint
	return store.editableBlock === props.block && store.activeBreakpoint === props.breakpoint;
});

const selectBlock = (e: MouseEvent | null) => {
	if (store.editableBlock === props.block || store.mode !== "select" || props.preview) {
		return;
	}
	store.selectBlock(props.block, e);
	store.activeBreakpoint = props.breakpoint;

	if (!props.preview) {
		store.leftPanelActiveTab = "Layers";
		store.rightPanelActiveTab = "Properties";
	}
};

const triggerContextMenu = (e: MouseEvent) => {
	if (props.block.isRoot() || isEditable.value) return;
	e.stopPropagation();
	e.preventDefault();
	selectBlock(e);
	nextTick(() => {
		editor.value?.element.dispatchEvent(new MouseEvent("contextmenu", e));
	});
};

const handleClick = (e: MouseEvent) => {
	if (isEditable.value) return;
	if (store.preventClick) {
		e.stopPropagation();
		e.preventDefault();
		store.preventClick = false;
		return;
	}
	selectBlock(e);
	e.stopPropagation();
	e.preventDefault();
};

const handleDoubleClick = (e: MouseEvent) => {
	if (isEditable.value) return;
	store.editableBlock = null;
	if (props.block.isText() || props.block.isLink() || props.block.isButton()) {
		store.editableBlock = props.block;
		e.stopPropagation();
	}

	// dblclick on container adds text block or selects text block if only one child
	let children = props.block.getChildren();
	if (props.block.isHTML()) {
		editor.value?.element.dispatchEvent(new MouseEvent("dblclick", e));
		e.stopPropagation();
	} else if (props.block.isContainer()) {
		if (!children.length) {
			const child = getBlockTemplate("text");
			props.block.setBaseStyle("alignItems", "center");
			props.block.setBaseStyle("justifyContent", "center");
			const childBlock = props.block.addChild(child);
			childBlock.makeBlockEditable();
		} else if (children.length === 1 && children[0].isText()) {
			const child = children[0];
			child.makeBlockEditable();
		}
		e.stopPropagation();
	}
};

const handleMouseOver = (e: MouseEvent) => {
	if (store.mode === "move") return;
	store.hoveredBlock = props.block.blockId;
	store.hoveredBreakpoint = props.breakpoint;
	e.stopPropagation();
};

const handleMouseLeave = (e: MouseEvent) => {
	if (store.mode === "move") return;
	if (store.hoveredBlock === props.block.blockId) {
		store.hoveredBlock = null;
		e.stopPropagation();
	}
};

const showBlock = computed(() => {
	// const data = props.block.getVisibilityCondition()
	// 	? getDataForKey(props.data, props.block.getVisibilityCondition() as string)
	// 	: true;
	return true;
});

if (!props.preview) {
	watch(
		() => store.hoveredBlock,
		(newValue, oldValue) => {
			if (newValue === props.block.blockId) {
				isHovered.value = true;
			} else if (oldValue === props.block.blockId) {
				isHovered.value = false;
			}
		},
	);
	watch(
		() => store.activeCanvas?.selectedBlockIds,
		() => {
			if (store.activeCanvas?.isSelected(props.block)) {
				isSelected.value = true;
			} else {
				isSelected.value = false;
			}
		},
		{
			deep: true,
			immediate: true,
		},
	);
}
</script>
