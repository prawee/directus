<template>
	<v-list
		ref="listComponent"
		large
		class="collections-navigation"
		:mandatory="false"
		@contextmenu.prevent.stop="activateContextMenu"
	>
		<template v-if="customNavItems && customNavItems.length > 0">
			<template v-for="(group, index) in customNavItems" :key="group.name">
				<template
					v-if="(group.name === undefined || group.name === null) && group.accordion === 'always_open' && index === 0"
				>
					<v-list-item v-for="navItem in group.items" :key="navItem.to" :to="navItem.to" query>
						<v-list-item-icon><v-icon :name="navItem.icon" :color="navItem.color" /></v-list-item-icon>
						<v-list-item-content>
							<v-text-overflow :text="navItem.name" />
						</v-list-item-content>
					</v-list-item>
				</template>
				<template v-else>
					<v-detail
						:model-value="group.accordion === 'always_open' || isActive(group.name)"
						:disabled="group.accordion === 'always_open'"
						:start-open="group.accordion === 'start_open'"
						:label="group.name || null"
						:class="{ empty: group.name === null || group.name === undefined }"
						@update:model-value="toggleActive(group.name)"
					>
						<v-list-item v-for="navItem in group.items" :key="navItem.to" :to="navItem.to" query>
							<v-list-item-icon><v-icon :name="navItem.icon" :color="navItem.color" /></v-list-item-icon>
							<v-list-item-content>
								<v-text-overflow :text="navItem.name" />
							</v-list-item-content>
						</v-list-item>
					</v-detail>
				</template>
			</template>
		</template>

		<v-list-item
			v-for="navItem in navItems"
			v-else
			:key="navItem.to"
			:to="navItem.to"
			query
			@contextmenu.prevent.stop="activateContextMenu($event, navItem.to)"
		>
			<v-list-item-icon><v-icon :name="navItem.icon" :color="navItem.color" /></v-list-item-icon>
			<v-list-item-content>
				<v-text-overflow :text="navItem.name" />
			</v-list-item-content>
		</v-list-item>

		<template v-if="bookmarks.length > 0">
			<v-divider />

			<navigation-bookmark v-for="bookmark of bookmarks" :key="bookmark.id" :bookmark="bookmark" />
		</template>

		<div v-if="!customNavItems && !navItems.length && !bookmarks.length" class="empty">
			<template v-if="searchQuery !== null">
				<em>{{ t('no_collections_found') }}</em>
			</template>
			<template v-else-if="isAdmin">
				<v-button full-width outlined dashed to="/settings/data-model/+">{{ t('create_collection') }}</v-button>
			</template>
			<template v-else>
				{{ t('no_collections_copy') }}
			</template>
		</div>

		<template v-if="hiddenShown">
			<v-divider />

			<v-list-item
				v-for="navItem in hiddenNavItems"
				:key="navItem.to"
				class="hidden-collection"
				:to="navItem.to"
				query
				@contextmenu.prevent.stop="activateContextMenu($event, navItem.to)"
			>
				<v-list-item-icon><v-icon :name="navItem.icon" :color="navItem.color" /></v-list-item-icon>
				<v-list-item-content>
					<v-text-overflow :text="navItem.name" />
				</v-list-item-content>
			</v-list-item>
		</template>

		<v-menu ref="contextMenu" show-arrow placement="bottom-start">
			<v-list>
				<v-list-item clickable @click="hiddenShown = !hiddenShown">
					<v-list-item-icon>
						<v-icon :name="hiddenShown ? 'visibility_off' : 'visibility'" />
					</v-list-item-icon>
					<v-list-item-content>
						<v-text-overflow :text="hiddenShown ? t('hide_hidden_collections') : t('show_hidden_collections')" />
					</v-list-item-content>
				</v-list-item>
				<v-list-item
					v-if="isAdmin && contextMenuTarget && contextMenuTarget.includes('/collections')"
					clickable
					:to="'/settings/data-model' + contextMenuTarget.replace('/collections', '')"
				>
					<v-list-item-icon>
						<v-icon name="list_alt" />
					</v-list-item-icon>
					<v-list-item-content>
						<v-text-overflow :text="t('edit_collection')" />
					</v-list-item-content>
				</v-list-item>
			</v-list>
		</v-menu>
	</v-list>
</template>

<script lang="ts">
import { useI18n } from 'vue-i18n';
import { defineComponent, computed, ref, watchEffect, onMounted, ComponentPublicInstance } from 'vue';
import useNavigation from '../composables/use-navigation';
import { usePresetsStore, useUserStore } from '@/stores/';
import { orderBy } from 'lodash';
import NavigationBookmark from './navigation-bookmark.vue';
import { useSearch } from '../composables/use-search';

export default defineComponent({
	components: { NavigationBookmark },
	setup() {
		const { t } = useI18n();

		const { searchQuery, visible } = useSearch();
		const listComponent = ref<ComponentPublicInstance>();

		const contextMenu = ref();
		const contextMenuTarget = ref<undefined | string>();

		const presetsStore = usePresetsStore();
		const userStore = useUserStore();
		const isAdmin = computed(() => userStore.currentUser?.role.admin_access === true);
		const { hiddenShown, customNavItems, navItems, activeGroups, hiddenNavItems } = useNavigation(searchQuery);

		const bookmarks = computed(() => {
			return orderBy(
				presetsStore.collectionPresets
					.filter((preset) => {
						return preset.bookmark !== null && preset.collection.startsWith('directus_') === false;
					})
					.filter(
						(preset) =>
							typeof preset.bookmark !== 'string' ||
							preset.bookmark.toLocaleLowerCase().includes(searchQuery?.value?.toLocaleLowerCase() || '')
					)
					.map((preset) => {
						let scope = 'global';
						if (preset.role) scope = 'role';
						if (preset.user) scope = 'user';

						return {
							...preset,
							to: `/collections/${preset.collection}?bookmark=${preset.id}`,
							scope,
						};
					}),
				['bookmark'],
				['asc']
			);
		});

		watchEffect(() => {
			visible.value = bookmarks.value.length + navItems.value.length;
		});

		onMounted(() => {
			const activeEl = listComponent.value?.$el.querySelector('.v-list-item.active.link');
			activeEl?.scrollIntoView({ block: 'center' });
		});

		return {
			t,
			navItems,
			bookmarks,
			customNavItems,
			isAdmin,
			activeGroups,
			isActive,
			toggleActive,
			contextMenu,
			hiddenShown,
			hiddenNavItems,
			searchQuery,
			listComponent,
			activateContextMenu,
			contextMenuTarget,
		};

		function isActive(name: string) {
			return activeGroups.value.includes(name);
		}

		function toggleActive(name: string) {
			if (activeGroups.value.includes(name)) {
				activeGroups.value = activeGroups.value.filter((current: string) => current !== name);
			} else {
				activeGroups.value.push(name);
			}
		}

		function activateContextMenu(event: PointerEvent, target?: string) {
			contextMenuTarget.value = target;
			contextMenu.value.activate(event);
		}
	},
});
</script>

<style lang="scss" scoped>
.group-name {
	padding-left: 8px;
	font-weight: 600;
}

.empty {
	.v-button {
		--v-button-color: var(--foreground-subdued);
		--v-button-background-color: var(--foreground-subdued);
		--v-button-background-color-hover: var(--primary);
	}
}

.collections-navigation {
	--v-list-min-height: calc(100% - 64px);

	.v-detail {
		:deep(.v-divider) {
			margin: 0px;
		}

		&:not(:first-child) :deep(.v-divider) {
			margin-top: 8px;
		}

		&.empty :deep(.v-divider) {
			margin-bottom: 8px;
		}
	}
}

.hidden-collection {
	--v-list-item-color: var(--foreground-subdued);
}
</style>
