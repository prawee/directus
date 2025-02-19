<template>
	<div class="presentation-links">
		<v-button
			v-for="(link, index) in linksParsed"
			:key="index"
			class="action"
			:class="[link.type]"
			:secondary="link.type !== 'primary'"
			:disabled="disabled"
			:icon="!link.label"
			:href="link.url"
		>
			<v-icon v-if="link.icon" left :name="link.icon" />
			<span v-if="link.label">{{ link.label }}</span>
		</v-button>
	</div>
</template>

<script lang="ts">
import { defineComponent, PropType, ref, inject, computed } from 'vue';
import { render } from 'micromustache';

type Link = {
	icon: string;
	label: string;
	type: string;
	url?: string;
};

export default defineComponent({
	props: {
		disabled: {
			type: Boolean,
			default: false,
		},
		links: {
			type: Array as PropType<Link[]>,
			default: null,
		},
	},
	setup(props) {
		const values = inject('values', ref<Record<string, any>>({}));

		const linksParsed = computed(() => {
			return props.links.map((link) => ({
				...link,
				url: render(link.url ?? '', values.value),
			}));
		});

		return {
			linksParsed,
		};
	},
});
</script>

<style lang="scss" scoped>
.presentation-links {
	display: flex;
	flex-wrap: wrap;
	gap: 8px;
}

.action {
	&.info {
		--v-button-background-color: var(--blue);
		--v-button-background-color-hover: var(--blue-125);
		--v-button-color: var(--blue-alt);
		--v-button-color-hover: var(--blue-alt);
	}

	&.success {
		--v-button-background-color: var(--success);
		--v-button-background-color-hover: var(--success-125);
		--v-button-color: var(--success-alt);
		--v-button-color-hover: var(--success-alt);
	}

	&.warning {
		--v-button-background-color: var(--warning);
		--v-button-background-color-hover: var(--warning-125);
		--v-button-color: var(--warning-alt);
		--v-button-color-hover: var(--warning-alt);
	}

	&.danger {
		--v-button-icon-color: var(--white);
		--v-button-background-color: var(--danger);
		--v-button-background-color-hover: var(--danger-125);
		--v-button-color: var(--danger-alt);
		--v-button-color-hover: var(--danger-alt);
	}
}
</style>
