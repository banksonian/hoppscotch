<template>
  <div class="flex flex-col" :class="[{ 'bg-primaryLight': dragging }]">
    <div
      class="flex items-center group"
      @dragover.prevent
      @drop.prevent="dropEvent"
      @dragover="dragging = true"
      @drop="dragging = false"
      @dragleave="dragging = false"
      @dragend="dragging = false"
    >
      <span
        class="cursor-pointer flex px-4 justify-center items-center"
        @click="toggleShowChildren()"
      >
        <SmartIcon
          class="svg-icons"
          :class="{ 'text-green-500': isSelected }"
          :name="getCollectionIcon"
        />
      </span>
      <span
        class="
          cursor-pointer
          flex flex-1
          min-w-0
          py-2
          pr-2
          transition
          group-hover:text-secondaryDark
        "
        @click="toggleShowChildren()"
      >
        <span class="truncate">
          {{ folder.name ? folder.name : folder.title }}
        </span>
      </span>
      <div class="flex">
        <ButtonSecondary
          v-tippy="{ theme: 'tooltip' }"
          svg="folder-plus"
          :title="$t('folder.new')"
          class="hidden group-hover:inline-flex"
          @click.native="$emit('add-folder', { folder, path: folderPath })"
        />
        <span>
          <tippy
            ref="options"
            interactive
            trigger="click"
            theme="popover"
            arrow
          >
            <template #trigger>
              <ButtonSecondary
                v-tippy="{ theme: 'tooltip' }"
                :title="$t('action.more')"
                svg="more-vertical"
              />
            </template>
            <SmartItem
              svg="folder-plus"
              :label="`${$t('folder.new')}`"
              @click.native="
                () => {
                  $emit('add-folder', { folder, path: folderPath })
                  $refs.options.tippy().hide()
                }
              "
            />
            <SmartItem
              svg="edit"
              :label="`${$t('action.edit')}`"
              @click.native="
                () => {
                  $emit('edit-folder', { folder, folderPath })
                  $refs.options.tippy().hide()
                }
              "
            />
            <SmartItem
              svg="trash-2"
              color="red"
              :label="`${$t('action.delete')}`"
              @click.native="
                () => {
                  confirmRemove = true
                  $refs.options.tippy().hide()
                }
              "
            />
          </tippy>
        </span>
      </div>
    </div>
    <div v-if="showChildren || isFiltered">
      <CollectionsGraphqlFolder
        v-for="(subFolder, subFolderIndex) in folder.folders"
        :key="`subFolder-${String(subFolderIndex)}`"
        class="border-l border-dividerLight ml-6"
        :picked="picked"
        :saving-mode="savingMode"
        :folder="subFolder"
        :folder-index="subFolderIndex"
        :folder-path="`${folderPath}/${String(subFolderIndex)}`"
        :collection-index="collectionIndex"
        :doc="doc"
        :is-filtered="isFiltered"
        @add-folder="$emit('add-folder', $event)"
        @edit-folder="$emit('edit-folder', $event)"
        @edit-request="$emit('edit-request', $event)"
        @select="$emit('select', $event)"
      />
      <CollectionsGraphqlRequest
        v-for="(request, index) in folder.requests"
        :key="`request-${String(index)}`"
        class="border-l border-dividerLight ml-6"
        :picked="picked"
        :saving-mode="savingMode"
        :request="request"
        :collection-index="collectionIndex"
        :folder-index="folderIndex"
        :folder-path="folderPath"
        :folder-name="folder.name"
        :request-index="index"
        :doc="doc"
        @edit-request="$emit('edit-request', $event)"
        @select="$emit('select', $event)"
      />
      <div
        v-if="
          folder.folders &&
          folder.folders.length === 0 &&
          folder.requests &&
          folder.requests.length === 0
        "
        class="
          border-l border-dividerLight
          flex flex-col
          text-secondaryLight
          ml-6
          p-4
          items-center
          justify-center
        "
      >
        <i class="opacity-75 pb-2 material-icons">folder_open</i>
        <span class="text-center">
          {{ $t("empty.folder") }}
        </span>
      </div>
    </div>
    <SmartConfirmModal
      :show="confirmRemove"
      :title="`${$t('confirm.remove_folder')}`"
      @hide-modal="confirmRemove = false"
      @resolve="removeFolder"
    />
  </div>
</template>

<script lang="ts">
import { defineComponent } from "@nuxtjs/composition-api"
import { removeGraphqlFolder, moveGraphqlRequest } from "~/newstore/collections"

export default defineComponent({
  name: "Folder",
  props: {
    picked: { type: Object, default: null },
    // Whether the request is in a selectable mode (activates 'select' event)
    savingMode: { type: Boolean, default: false },
    folder: { type: Object, default: () => {} },
    folderIndex: { type: Number, default: null },
    collectionIndex: { type: Number, default: null },
    folderPath: { type: String, default: null },
    doc: Boolean,
    isFiltered: Boolean,
  },
  data() {
    return {
      showChildren: false,
      dragging: false,
      confirmRemove: false,
    }
  },
  computed: {
    isSelected(): boolean {
      return (
        this.picked &&
        this.picked.pickedType === "gql-my-folder" &&
        this.picked.folderPath === this.folderPath
      )
    },
    getCollectionIcon() {
      if (this.isSelected) return "check-circle"
      else if (!this.showChildren && !this.isFiltered) return "folder"
      else if (this.showChildren || this.isFiltered) return "folder-minus"
      else return "folder"
    },
  },
  methods: {
    pick() {
      this.$emit("select", {
        picked: {
          pickedType: "gql-my-folder",
          folderPath: this.folderPath,
        },
      })
    },
    toggleShowChildren() {
      if (this.savingMode) {
        this.pick()
      }

      this.showChildren = !this.showChildren
    },
    removeFolder() {
      // Cancel pick if the picked folder is deleted
      if (
        this.picked &&
        this.picked.pickedType === "gql-my-folder" &&
        this.picked.folderPath === this.folderPath
      ) {
        this.$emit("select", { picked: null })
      }

      removeGraphqlFolder(this.folderPath)
      this.$toast.success(`${this.$t("state.deleted")}`, {
        icon: "delete",
      })
    },
    dropEvent({ dataTransfer }: any) {
      this.dragging = !this.dragging
      const folderPath = dataTransfer.getData("folderPath")
      const requestIndex = dataTransfer.getData("requestIndex")

      moveGraphqlRequest(folderPath, requestIndex, this.folderPath)
    },
  },
})
</script>
