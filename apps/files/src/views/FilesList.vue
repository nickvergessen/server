<!--
  - SPDX-FileCopyrightText: 2023 Nextcloud GmbH and Nextcloud contributors
  - SPDX-License-Identifier: AGPL-3.0-or-later
-->
<template>
	<NcAppContent :page-heading="pageHeading" data-cy-files-content>
		<div class="files-list__header">
			<!-- Current folder breadcrumbs -->
			<BreadCrumbs :path="dir" @reload="fetchContent">
				<template #actions>
					<!-- Sharing button -->
					<NcButton v-if="canShare && filesListWidth >= 512"
						:aria-label="shareButtonLabel"
						:class="{ 'files-list__header-share-button--shared': shareButtonType }"
						:title="shareButtonLabel"
						class="files-list__header-share-button"
						type="tertiary"
						@click="openSharingSidebar">
						<template #icon>
							<LinkIcon v-if="shareButtonType === Type.SHARE_TYPE_LINK" />
							<AccountPlusIcon v-else :size="20" />
						</template>
					</NcButton>

					<!-- Disabled upload button -->
					<NcButton v-if="!canUpload || isQuotaExceeded"
						:aria-label="cantUploadLabel"
						:title="cantUploadLabel"
						class="files-list__header-upload-button--disabled"
						:disabled="true"
						type="secondary">
						<template #icon>
							<PlusIcon :size="20" />
						</template>
						{{ t('files', 'New') }}
					</NcButton>

					<!-- Uploader -->
					<UploadPicker v-else-if="currentFolder"
						allow-folders
						class="files-list__header-upload-button"
						:content="getContent"
						:destination="currentFolder"
						:forbidden-characters="forbiddenCharacters"
						multiple
						@failed="onUploadFail"
						@uploaded="onUpload" />
				</template>
			</BreadCrumbs>

			<NcButton v-if="filesListWidth >= 512 && enableGridView"
				:aria-label="gridViewButtonLabel"
				:title="gridViewButtonLabel"
				class="files-list__header-grid-button"
				type="tertiary"
				@click="toggleGridView">
				<template #icon>
					<ListViewIcon v-if="userConfig.grid_view" />
					<ViewGridIcon v-else />
				</template>
			</NcButton>

			<!-- Secondary loading indicator -->
			<NcLoadingIcon v-if="isRefreshing" class="files-list__refresh-icon" />
		</div>

		<!-- Drag and drop notice -->
		<DragAndDropNotice v-if="!loading && canUpload" :current-folder="currentFolder" />

		<!-- Initial loading -->
		<NcLoadingIcon v-if="loading && !isRefreshing"
			class="files-list__loading-icon"
			:size="38"
			:name="t('files', 'Loading current folder')" />

		<!-- Empty content placeholder -->
		<NcEmptyContent v-else-if="!loading && isEmptyDir"
			:name="currentView?.emptyTitle || t('files', 'No files in here')"
			:description="currentView?.emptyCaption || t('files', 'Upload some content or sync with your devices!')"
			data-cy-files-content-empty>
			<template v-if="dir !== '/'" #action>
				<!-- Uploader -->
				<UploadPicker v-if="currentFolder && canUpload && !isQuotaExceeded"
					allow-folders
					class="files-list__header-upload-button"
					:content="getContent"
					:destination="currentFolder"
					:forbidden-characters="forbiddenCharacters"
					multiple
					@failed="onUploadFail"
					@uploaded="onUpload" />
				<NcButton v-else
					:aria-label="t('files', 'Go to the previous folder')"
					:to="toPreviousDir"
					type="primary">
					{{ t('files', 'Go back') }}
				</NcButton>
			</template>
			<template #icon>
				<NcIconSvgWrapper :svg="currentView.icon" />
			</template>
		</NcEmptyContent>

		<!-- File list -->
		<FilesListVirtual v-else
			ref="filesListVirtual"
			:current-folder="currentFolder"
			:current-view="currentView"
			:nodes="dirContentsSorted" />
	</NcAppContent>
</template>

<script lang="ts">
import type { ContentsWithRoot } from '@nextcloud/files'
import type { Upload } from '@nextcloud/upload'
import type { CancelablePromise } from 'cancelable-promise'
import type { ComponentPublicInstance } from 'vue'
import type { Route } from 'vue-router'
import type { UserConfig } from '../types.ts'

import { getCapabilities } from '@nextcloud/capabilities'
import { emit, subscribe, unsubscribe } from '@nextcloud/event-bus'
import { Folder, Node, Permission } from '@nextcloud/files'
import { translate as t } from '@nextcloud/l10n'
import { join, dirname, normalize } from 'path'
import { showError, showWarning } from '@nextcloud/dialogs'
import { Type } from '@nextcloud/sharing'
import { UploadPicker, UploadStatus } from '@nextcloud/upload'
import { loadState } from '@nextcloud/initial-state'
import { defineComponent } from 'vue'

import LinkIcon from 'vue-material-design-icons/Link.vue'
import ListViewIcon from 'vue-material-design-icons/FormatListBulletedSquare.vue'
import NcAppContent from '@nextcloud/vue/dist/Components/NcAppContent.js'
import NcButton from '@nextcloud/vue/dist/Components/NcButton.js'
import NcEmptyContent from '@nextcloud/vue/dist/Components/NcEmptyContent.js'
import NcIconSvgWrapper from '@nextcloud/vue/dist/Components/NcIconSvgWrapper.js'
import NcLoadingIcon from '@nextcloud/vue/dist/Components/NcLoadingIcon.js'
import PlusIcon from 'vue-material-design-icons/Plus.vue'
import AccountPlusIcon from 'vue-material-design-icons/AccountPlus.vue'
import ViewGridIcon from 'vue-material-design-icons/ViewGrid.vue'

import { action as sidebarAction } from '../actions/sidebarAction.ts'
import { useNavigation } from '../composables/useNavigation.ts'
import { useFilesStore } from '../store/files.ts'
import { usePathsStore } from '../store/paths.ts'
import { useSelectionStore } from '../store/selection.ts'
import { useUploaderStore } from '../store/uploader.ts'
import { useUserConfigStore } from '../store/userconfig.ts'
import { useViewConfigStore } from '../store/viewConfig.ts'
import { orderBy } from '../services/SortingService.ts'
import BreadCrumbs from '../components/BreadCrumbs.vue'
import FilesListVirtual from '../components/FilesListVirtual.vue'
import filesListWidthMixin from '../mixins/filesListWidth.ts'
import filesSortingMixin from '../mixins/filesSorting.ts'
import logger from '../logger.js'
import DragAndDropNotice from '../components/DragAndDropNotice.vue'
import debounce from 'debounce'

const isSharingEnabled = (getCapabilities() as { files_sharing?: boolean })?.files_sharing !== undefined

export default defineComponent({
	name: 'FilesList',

	components: {
		BreadCrumbs,
		DragAndDropNotice,
		FilesListVirtual,
		LinkIcon,
		ListViewIcon,
		NcAppContent,
		NcButton,
		NcEmptyContent,
		NcIconSvgWrapper,
		NcLoadingIcon,
		PlusIcon,
		AccountPlusIcon,
		UploadPicker,
		ViewGridIcon,
	},

	mixins: [
		filesListWidthMixin,
		filesSortingMixin,
	],

	setup() {
		const filesStore = useFilesStore()
		const pathsStore = usePathsStore()
		const selectionStore = useSelectionStore()
		const uploaderStore = useUploaderStore()
		const userConfigStore = useUserConfigStore()
		const viewConfigStore = useViewConfigStore()
		const { currentView } = useNavigation()

		const enableGridView = (loadState('core', 'config', [])['enable_non-accessible_features'] ?? true)
		const forbiddenCharacters = loadState<string[]>('files', 'forbiddenCharacters', [])

		return {
			currentView,

			filesStore,
			pathsStore,
			selectionStore,
			uploaderStore,
			userConfigStore,
			viewConfigStore,

			// non reactive data
			enableGridView,
			forbiddenCharacters,
			Type,
		}
	},

	data() {
		return {
			filterText: '',
			loading: true,
			promise: null as CancelablePromise<ContentsWithRoot> | Promise<ContentsWithRoot> | null,

			unsubscribeStoreCallback: () => {},
		}
	},

	computed: {
		/**
		 * Handle search event from unified search.
		 */
		onSearch() {
			return debounce((searchEvent: { query: string }) => {
				console.debug('Files app handling search event from unified search...', searchEvent)
				this.filterText = searchEvent.query
			}, 500)
		},

		/**
		 * Get a callback function for the uploader to fetch directory contents for conflict resolution
		 */
		getContent() {
			const view = this.currentView
			return async (path?: string) => {
				// as the path is allowed to be undefined we need to normalize the path ('//' to '/')
				const normalizedPath = normalize(`${this.currentFolder?.path ?? ''}/${path ?? ''}`)
				// use the current view to fetch the content for the requested path
				return (await view.getContents(normalizedPath)).contents
			}
		},

		userConfig(): UserConfig {
			return this.userConfigStore.userConfig
		},

		pageHeading(): string {
			return this.currentView?.name ?? t('files', 'Files')
		},

		/**
		 * The current directory query.
		 */
		dir(): string {
			// Remove any trailing slash but leave root slash
			return (this.$route?.query?.dir?.toString() || '/').replace(/^(.+)\/$/, '$1')
		},

		/**
		 * The current file id
		 */
		fileId(): number | null {
			const number = Number.parseInt(this.$route?.params.fileid ?? '')
			return Number.isNaN(number) ? null : number
		},

		/**
		 * The current folder.
		 */
		currentFolder(): Folder | undefined {
			if (!this.currentView?.id) {
				return
			}

			if (this.dir === '/') {
				return this.filesStore.getRoot(this.currentView.id)
			}

			const source = this.pathsStore.getPath(this.currentView.id, this.dir)
			if (source === undefined) {
				return
			}

			return this.filesStore.getNode(source) as Folder
		},

		/**
		 * Directory content sorting parameters
		 * Provided by an extra computed property for caching
		 */
		sortingParameters() {
			const identifiers = [
				// 1: Sort favorites first if enabled
				...(this.userConfig.sort_favorites_first ? [v => v.attributes?.favorite !== 1] : []),
				// 2: Sort folders first if sorting by name
				...(this.userConfig.sort_folders_first ? [v => v.type !== 'folder'] : []),
				// 3: Use sorting mode if NOT basename (to be able to use displayName too)
				...(this.sortingMode !== 'basename' ? [v => v[this.sortingMode]] : []),
				// 4: Use displayname if available, fallback to name
				v => v.attributes?.displayname || v.basename,
				// 5: Finally, use basename if all previous sorting methods failed
				v => v.basename,
			]
			const orders = [
				// (for 1): always sort favorites before normal files
				...(this.userConfig.sort_favorites_first ? ['asc'] : []),
				// (for 2): always sort folders before files
				...(this.userConfig.sort_folders_first ? ['asc'] : []),
				// (for 3): Reverse if sorting by mtime as mtime higher means edited more recent -> lower
				...(this.sortingMode === 'mtime' ? [this.isAscSorting ? 'desc' : 'asc'] : []),
				// (also for 3 so make sure not to conflict with 2 and 3)
				...(this.sortingMode !== 'mtime' && this.sortingMode !== 'basename' ? [this.isAscSorting ? 'asc' : 'desc'] : []),
				// for 4: use configured sorting direction
				this.isAscSorting ? 'asc' : 'desc',
				// for 5: use configured sorting direction
				this.isAscSorting ? 'asc' : 'desc',
			] as ('asc'|'desc')[]
			return [identifiers, orders] as const
		},

		/**
		 * The current directory contents.
		 */
		dirContentsSorted(): Node[] {
			if (!this.currentView) {
				return []
			}

			let filteredDirContent = [...this.dirContents]
			// Filter based on the filterText obtained from nextcloud:unified-search.search event.
			if (this.filterText) {
				filteredDirContent = filteredDirContent.filter(node => {
					return node.basename.toLowerCase().includes(this.filterText.toLowerCase())
				})
				console.debug('Files view filtered', filteredDirContent)
			}

			const customColumn = (this.currentView?.columns || [])
				.find(column => column.id === this.sortingMode)

			// Custom column must provide their own sorting methods
			if (customColumn?.sort && typeof customColumn.sort === 'function') {
				const results = [...this.dirContents].sort(customColumn.sort)
				return this.isAscSorting ? results : results.reverse()
			}

			return orderBy(
				filteredDirContent,
				...this.sortingParameters,
			)
		},

		dirContents(): Node[] {
			const showHidden = this.userConfigStore?.userConfig.show_hidden
			return (this.currentFolder?._children || [])
				.map(this.getNode)
				.filter(file => {
					if (!showHidden) {
						return file && file?.attributes?.hidden !== true && !file?.basename.startsWith('.')
					}

					return !!file
				})
		},

		/**
		 * The current directory is empty.
		 */
		isEmptyDir(): boolean {
			return this.dirContents.length === 0
		},

		/**
		 * We are refreshing the current directory.
		 * But we already have a cached version of it
		 * that is not empty.
		 */
		isRefreshing(): boolean {
			return this.currentFolder !== undefined
				&& !this.isEmptyDir
				&& this.loading
		},

		/**
		 * Route to the previous directory.
		 */
		toPreviousDir(): Route {
			const dir = this.dir.split('/').slice(0, -1).join('/') || '/'
			return { ...this.$route, query: { dir } }
		},

		shareAttributes(): number[] | undefined {
			if (!this.currentFolder?.attributes?.['share-types']) {
				return undefined
			}
			return Object.values(this.currentFolder?.attributes?.['share-types'] || {}).flat() as number[]
		},
		shareButtonLabel() {
			if (!this.shareAttributes) {
				return t('files', 'Share')
			}

			if (this.shareButtonType === Type.SHARE_TYPE_LINK) {
				return t('files', 'Shared by link')
			}
			return t('files', 'Shared')
		},
		shareButtonType(): Type | null {
			if (!this.shareAttributes) {
				return null
			}

			// If all types are links, show the link icon
			if (this.shareAttributes.some(type => type === Type.SHARE_TYPE_LINK)) {
				return Type.SHARE_TYPE_LINK
			}

			return Type.SHARE_TYPE_USER
		},

		gridViewButtonLabel() {
			return this.userConfig.grid_view
				? t('files', 'Switch to list view')
				: t('files', 'Switch to grid view')
		},

		/**
		 * Check if the current folder has create permissions
		 */
		canUpload() {
			return this.currentFolder && (this.currentFolder.permissions & Permission.CREATE) !== 0
		},
		isQuotaExceeded() {
			return this.currentFolder?.attributes?.['quota-available-bytes'] === 0
		},
		cantUploadLabel() {
			if (this.isQuotaExceeded) {
				return t('files', 'Your have used your space quota and cannot upload files anymore')
			}
			return t('files', 'You don’t have permission to upload or create files here')
		},

		/**
		 * Check if current folder has share permissions
		 */
		canShare() {
			return isSharingEnabled
				&& this.currentFolder && (this.currentFolder.permissions & Permission.SHARE) !== 0
		},
	},

	watch: {
		currentView(newView, oldView) {
			if (newView?.id === oldView?.id) {
				return
			}

			logger.debug('View changed', { newView, oldView })
			this.selectionStore.reset()
			this.triggerResetSearch()
			this.fetchContent()
		},

		dir(newDir, oldDir) {
			logger.debug('Directory changed', { newDir, oldDir })
			// TODO: preserve selection on browsing?
			this.selectionStore.reset()
			this.triggerResetSearch()
			this.fetchContent()

			// Scroll to top, force virtual scroller to re-render
			const filesListVirtual = this.$refs?.filesListVirtual as ComponentPublicInstance<typeof FilesListVirtual> | undefined
			if (filesListVirtual?.$el) {
				filesListVirtual.$el.scrollTop = 0
			}
		},

		dirContents(contents) {
			logger.debug('Directory contents changed', { view: this.currentView, folder: this.currentFolder, contents })
			emit('files:list:updated', { view: this.currentView, folder: this.currentFolder, contents })
		},
	},

	mounted() {
		this.fetchContent()

		subscribe('files:node:deleted', this.onNodeDeleted)
		subscribe('files:node:updated', this.onUpdatedNode)
		subscribe('nextcloud:unified-search:search', this.onSearch)
		subscribe('nextcloud:unified-search:reset', this.onResetSearch)

		// reload on settings change
		this.unsubscribeStoreCallback = this.userConfigStore.$subscribe(() => this.fetchContent(), { deep: true })
	},

	unmounted() {
		unsubscribe('files:node:deleted', this.onNodeDeleted)
		unsubscribe('files:node:updated', this.onUpdatedNode)
		unsubscribe('nextcloud:unified-search:search', this.onSearch)
		unsubscribe('nextcloud:unified-search:reset', this.onResetSearch)
		this.unsubscribeStoreCallback()
	},

	methods: {
		t,

		async fetchContent() {
			this.loading = true
			const dir = this.dir
			const currentView = this.currentView

			if (!currentView) {
				logger.debug('The current view doesn\'t exists or is not ready.', { currentView })
				return
			}

			// If we have a cancellable promise ongoing, cancel it
			if (this.promise && 'cancel' in this.promise) {
				this.promise.cancel()
				logger.debug('Cancelled previous ongoing fetch')
			}

			// Fetch the current dir contents
			this.promise = currentView.getContents(dir) as Promise<ContentsWithRoot>
			try {
				const { folder, contents } = await this.promise
				logger.debug('Fetched contents', { dir, folder, contents })

				// Update store
				this.filesStore.updateNodes(contents)

				// Define current directory children
				// TODO: make it more official
				this.$set(folder, '_children', contents.map(node => node.source))

				// If we're in the root dir, define the root
				if (dir === '/') {
					this.filesStore.setRoot({ service: currentView.id, root: folder })
				} else {
					// Otherwise, add the folder to the store
					if (folder.fileid) {
						this.filesStore.updateNodes([folder])
						this.pathsStore.addPath({ service: currentView.id, source: folder.source, path: dir })
					} else {
						// If we're here, the view API messed up
						logger.fatal('Invalid root folder returned', { dir, folder, currentView })
					}
				}

				// Update paths store
				const folders = contents.filter(node => node.type === 'folder')
				folders.forEach((node) => {
					this.pathsStore.addPath({ service: currentView.id, source: node.source, path: join(dir, node.basename) })
				})
			} catch (error) {
				logger.error('Error while fetching content', { error })
			} finally {
				this.loading = false
			}

		},

		/**
		 * Get a cached note from the store
		 *
		 * @param {number} fileId the file id to get
		 * @return {Folder|File}
		 */
		getNode(fileId) {
			return this.filesStore.getNode(fileId)
		},

		/**
		 * Handle the node deleted event to reset open file
		 * @param node The deleted node
		 */
		 onNodeDeleted(node: Node) {
			if (node.fileid && node.fileid === this.fileId) {
				if (node.fileid === this.currentFolder?.fileid) {
					// Handle the edge case that the current directory is deleted
					// in this case we neeed to keept the current view but move to the parent directory
					window.OCP.Files.Router.goToRoute(
						null,
						{ view: this.$route.params.view },
						{ dir: this.currentFolder?.dirname ?? '/' },
					)
				} else {
					// If the currently active file is deleted we need to remove the fileid and possible the `openfile` query
					window.OCP.Files.Router.goToRoute(
						null,
						{ ...this.$route.params, fileid: undefined },
						{ ...this.$route.query, openfile: undefined },
					)
				}
			}
		},

		/**
		 * The upload manager have finished handling the queue
		 * @param {Upload} upload the uploaded data
		 */
		onUpload(upload: Upload) {
			// Let's only refresh the current Folder
			// Navigating to a different folder will refresh it anyway
			const needsRefresh = dirname(upload.source) === this.currentFolder!.source

			// TODO: fetch uploaded files data only
			// Use parseInt(upload.response?.headers?.['oc-fileid']) to get the fileid
			if (needsRefresh) {
				// fetchContent will cancel the previous ongoing promise
				this.fetchContent()
			}
		},

		async onUploadFail(upload: Upload) {
			const status = upload.response?.status || 0

			if (upload.status === UploadStatus.CANCELLED) {
				showWarning(t('files', 'Upload was cancelled by user'))
				return
			}

			// Check known status codes
			if (status === 507) {
				showError(t('files', 'Not enough free space'))
				return
			} else if (status === 404 || status === 409) {
				showError(t('files', 'Target folder does not exist any more'))
				return
			} else if (status === 403) {
				showError(t('files', 'Operation is blocked by access control'))
				return
			}

			// Else we try to parse the response error message
			if (typeof upload.response?.data === 'string') {
				try {
					const parser = new DOMParser()
					const doc = parser.parseFromString(upload.response.data, 'text/xml')
					const message = doc.getElementsByTagName('s:message')[0]?.textContent ?? ''
					if (message.trim() !== '') {
						// The server message is also translated
						showError(t('files', 'Error during upload: {message}', { message }))
						return
					}
				} catch (error) {
					logger.error('Could not parse message', { error })
				}
			}

			// Finally, check the status code if we have one
			if (status !== 0) {
				showError(t('files', 'Error during upload, status code {status}', { status }))
				return
			}

			showError(t('files', 'Unknown error during upload'))
		},

		/**
		 * Refreshes the current folder on update.
		 *
		 * @param node is the file/folder being updated.
		 */
		onUpdatedNode(node?: Node) {
			if (node?.fileid === this.currentFolder?.fileid) {
				this.fetchContent()
			}
		},

		/**
		 * Handle reset search query event
		 */
		onResetSearch() {
			// Reset debounced calls to not set the query again
			this.onSearch.clear()
			// Reset filter query
			this.filterText = ''
		},

		/**
		 * Trigger a reset of the local search (part of unified search)
		 * This is usful to reset the search on directory / view change
		 */
		triggerResetSearch() {
			emit('nextcloud:unified-search:reset')
		},

		openSharingSidebar() {
			if (!this.currentFolder) {
				logger.debug('No current folder found for opening sharing sidebar')
				return
			}

			if (window?.OCA?.Files?.Sidebar?.setActiveTab) {
				window.OCA.Files.Sidebar.setActiveTab('sharing')
			}
			sidebarAction.exec(this.currentFolder, this.currentView!, this.currentFolder.path)
		},
		toggleGridView() {
			this.userConfigStore.update('grid_view', !this.userConfig.grid_view)
		},
	},
})
</script>

<style scoped lang="scss">
.app-content {
	// Virtual list needs to be full height and is scrollable
	display: flex;
	overflow: hidden;
	flex-direction: column;
	max-height: 100%;
	position: relative !important;
}

.files-list {
	&__header {
		display: flex;
		align-items: center;
		// Do not grow or shrink (vertically)
		flex: 0 0;
		max-width: 100%;
		// Align with the navigation toggle icon
		margin-block: var(--app-navigation-padding, 4px);
		margin-inline: calc(var(--default-clickable-area, 44px) + 2 * var(--app-navigation-padding, 4px)) var(--app-navigation-padding, 4px);

		>* {
			// Do not grow or shrink (horizontally)
			// Only the breadcrumbs shrinks
			flex: 0 0;
		}

		&-share-button {
			color: var(--color-text-maxcontrast) !important;

			&--shared {
				color: var(--color-main-text) !important;
			}
		}
	}

	&__refresh-icon {
		flex: 0 0 44px;
		width: 44px;
		height: 44px;
	}

	&__loading-icon {
		margin: auto;
	}
}
</style>
