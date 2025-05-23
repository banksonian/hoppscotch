<template>
  <Splitpanes
    class="smart-splitter"
    :dbl-click-splitter="false"
    :horizontal="!(windowInnerWidth.x.value >= 768)"
  >
    <Pane class="hide-scrollbar !overflow-auto">
      <Splitpanes
        class="smart-splitter"
        :dbl-click-splitter="false"
        :horizontal="COLUMN_LAYOUT"
      >
        <Pane class="hide-scrollbar !overflow-auto">
          <AppSection label="request">
            <div class="bg-primary flex p-4 top-0 z-10 sticky">
              <div class="space-x-2 flex-1 inline-flex">
                <input
                  id="websocket-url"
                  v-model="url"
                  v-focus
                  class="
                    bg-primaryLight
                    border border-divider
                    rounded
                    text-secondaryDark
                    w-full
                    py-2
                    px-4
                    hover:border-dividerDark
                    focus-visible:bg-transparent
                    focus-visible:border-dividerDark
                  "
                  type="url"
                  autocomplete="off"
                  spellcheck="false"
                  :class="{ error: !urlValid }"
                  :placeholder="$t('websocket.url')"
                  @keyup.enter="urlValid ? toggleConnection() : null"
                />
                <ButtonPrimary
                  id="connect"
                  :disabled="!urlValid"
                  class="w-32"
                  name="connect"
                  :label="
                    !connectionState
                      ? $t('action.connect')
                      : $t('action.disconnect')
                  "
                  :loading="connectingState"
                  @click.native="toggleConnection"
                />
              </div>
            </div>
            <div
              class="
                bg-primary
                border-b border-dividerLight
                flex flex-1
                top-upperPrimaryStickyFold
                pl-4
                z-10
                sticky
                items-center
                justify-between
              "
            >
              <label class="font-semibold text-secondaryLight">
                {{ $t("websocket.protocols") }}
              </label>
              <div class="flex">
                <ButtonSecondary
                  v-tippy="{ theme: 'tooltip' }"
                  :title="$t('action.clear_all')"
                  svg="trash-2"
                  @click.native="clearContent"
                />
                <ButtonSecondary
                  v-tippy="{ theme: 'tooltip' }"
                  :title="$t('add.new')"
                  svg="plus"
                  @click.native="addProtocol"
                />
              </div>
            </div>
            <div
              v-for="(protocol, index) of protocols"
              :key="`protocol-${index}`"
              class="
                divide-x divide-dividerLight
                border-b border-dividerLight
                flex
              "
            >
              <input
                v-model="protocol.value"
                class="bg-transparent flex flex-1 py-2 px-4"
                :placeholder="$t('count.protocol', { count: index + 1 })"
                name="message"
                type="text"
                autocomplete="off"
              />
              <span>
                <ButtonSecondary
                  v-tippy="{ theme: 'tooltip' }"
                  :title="
                    protocol.hasOwnProperty('active')
                      ? protocol.active
                        ? $t('action.turn_off')
                        : $t('action.turn_on')
                      : $t('action.turn_off')
                  "
                  :svg="
                    protocol.hasOwnProperty('active')
                      ? protocol.active
                        ? 'check-circle'
                        : 'circle'
                      : 'check-circle'
                  "
                  color="green"
                  @click.native="
                    protocol.active = protocol.hasOwnProperty('active')
                      ? !protocol.active
                      : false
                  "
                />
              </span>
              <span>
                <ButtonSecondary
                  v-tippy="{ theme: 'tooltip' }"
                  :title="$t('action.remove')"
                  svg="trash"
                  color="red"
                  @click.native="deleteProtocol({ index })"
                />
              </span>
            </div>
            <div
              v-if="protocols.length === 0"
              class="
                flex flex-col
                text-secondaryLight
                p-4
                items-center
                justify-center
              "
            >
              <i class="opacity-75 pb-2 material-icons">topic</i>
              <span class="text-center">
                {{ $t("empty.protocols") }}
              </span>
            </div>
          </AppSection>
        </Pane>
        <Pane class="hide-scrollbar !overflow-auto">
          <AppSection label="response">
            <RealtimeLog
              :title="$t('websocket.log')"
              :log="communication.log"
            />
          </AppSection>
        </Pane>
      </Splitpanes>
    </Pane>
    <Pane
      v-if="RIGHT_SIDEBAR"
      max-size="35"
      size="25"
      min-size="20"
      class="hide-scrollbar !overflow-auto"
    >
      <AppSection label="messages">
        <div class="flex flex-col flex-1 p-4 inline-flex">
          <label
            for="websocket-message"
            class="font-semibold text-secondaryLight"
          >
            {{ $t("websocket.communication") }}
          </label>
        </div>
        <div class="flex space-x-2 px-4">
          <input
            id="websocket-message"
            v-model="communication.input"
            name="message"
            type="text"
            autocomplete="off"
            :disabled="!connectionState"
            :placeholder="$t('websocket.message')"
            class="input"
            @keyup.enter="connectionState ? sendMessage() : null"
            @keyup.up="connectionState ? walkHistory('up') : null"
            @keyup.down="connectionState ? walkHistory('down') : null"
          />
          <ButtonPrimary
            id="send"
            name="send"
            :disabled="!connectionState"
            :label="$t('action.send')"
            @click.native="sendMessage"
          />
        </div>
      </AppSection>
    </Pane>
  </Splitpanes>
</template>

<script>
import { defineComponent } from "@nuxtjs/composition-api"
import { Splitpanes, Pane } from "splitpanes"
import "splitpanes/dist/splitpanes.css"
import debounce from "lodash/debounce"
import { logHoppRequestRunToAnalytics } from "~/helpers/fb/analytics"
import useWindowSize from "~/helpers/utils/useWindowSize"
import { useSetting } from "~/newstore/settings"

export default defineComponent({
  components: { Splitpanes, Pane },
  setup() {
    return {
      windowInnerWidth: useWindowSize(),
      RIGHT_SIDEBAR: useSetting("RIGHT_SIDEBAR"),
      COLUMN_LAYOUT: useSetting("COLUMN_LAYOUT"),
    }
  },
  data() {
    return {
      connectionState: false,
      connectingState: false,
      url: "wss://echo.websocket.org",
      isUrlValid: true,
      socket: null,
      communication: {
        log: null,
        input: "",
      },
      currentIndex: -1, // index of the message log array to put in input box
      protocols: [],
      activeProtocols: [],
    }
  },
  computed: {
    urlValid() {
      return this.isUrlValid
    },
  },
  watch: {
    url() {
      this.debouncer()
    },
    protocols: {
      handler(newVal) {
        this.activeProtocols = newVal
          .filter((item) =>
            Object.prototype.hasOwnProperty.call(item, "active")
              ? item.active === true
              : true
          )
          .map(({ value }) => value)
      },
      deep: true,
    },
  },
  mounted() {
    if (process.browser) {
      this.worker = this.$worker.createRejexWorker()
      this.worker.addEventListener("message", this.workerResponseHandler)
    }
  },
  destroyed() {
    this.worker.terminate()
  },
  methods: {
    clearContent() {
      this.protocols = []
    },
    debouncer: debounce(function () {
      this.worker.postMessage({ type: "ws", url: this.url })
    }, 1000),
    workerResponseHandler({ data }) {
      if (data.url === this.url) this.isUrlValid = data.result
    },
    toggleConnection() {
      // If it is connecting:
      if (!this.connectionState) return this.connect()
      // Otherwise, it's disconnecting.
      else return this.disconnect()
    },
    connect() {
      this.communication.log = [
        {
          payload: this.$t("state.connecting_to", { name: this.url }),
          source: "info",
          color: "var(--accent-color)",
        },
      ]
      try {
        this.connectingState = true
        this.socket = new WebSocket(this.url, this.activeProtocols)
        this.socket.onopen = () => {
          this.connectingState = false
          this.connectionState = true
          this.communication.log = [
            {
              payload: this.$t("state.connected_to", { name: this.url }),
              source: "info",
              color: "var(--accent-color)",
              ts: new Date().toLocaleTimeString(),
            },
          ]
          this.$toast.success(this.$t("state.connected"), {
            icon: "sync",
          })
        }
        this.socket.onerror = () => {
          this.handleError()
        }
        this.socket.onclose = () => {
          this.connectionState = false
          this.communication.log.push({
            payload: this.$t("state.disconnected_from", { name: this.url }),
            source: "info",
            color: "#ff5555",
            ts: new Date().toLocaleTimeString(),
          })
          this.$toast.error(this.$t("state.disconnected"), {
            icon: "sync_disabled",
          })
        }
        this.socket.onmessage = ({ data }) => {
          this.communication.log.push({
            payload: data,
            source: "server",
            ts: new Date().toLocaleTimeString(),
          })
        }
      } catch (e) {
        this.handleError(e)
        this.$toast.error(this.$t("error.something_went_wrong"), {
          icon: "error_outline",
        })
      }

      logHoppRequestRunToAnalytics({
        platform: "wss",
      })
    },
    disconnect() {
      if (this.socket) {
        this.socket.close()
        this.connectionState = false
        this.connectingState = false
      }
    },
    handleError(error) {
      this.disconnect()
      this.connectionState = false
      this.communication.log.push({
        payload: this.$t("error.something_went_wrong"),
        source: "info",
        color: "#ff5555",
        ts: new Date().toLocaleTimeString(),
      })
      if (error !== null)
        this.communication.log.push({
          payload: error,
          source: "info",
          color: "#ff5555",
          ts: new Date().toLocaleTimeString(),
        })
    },
    sendMessage() {
      const message = this.communication.input
      this.socket.send(message)
      this.communication.log.push({
        payload: message,
        source: "client",
        ts: new Date().toLocaleTimeString(),
      })
      this.communication.input = ""
    },
    walkHistory(direction) {
      const clientMessages = this.communication.log.filter(
        ({ source }) => source === "client"
      )
      const length = clientMessages.length
      switch (direction) {
        case "up":
          if (length > 0 && this.currentIndex !== 0) {
            // does nothing if message log is empty or the currentIndex is 0 when up arrow is pressed
            if (this.currentIndex === -1) {
              this.currentIndex = length - 1
              this.communication.input =
                clientMessages[this.currentIndex].payload
            } else if (this.currentIndex === 0) {
              this.communication.input = clientMessages[0].payload
            } else if (this.currentIndex > 0) {
              this.currentIndex = this.currentIndex - 1
              this.communication.input =
                clientMessages[this.currentIndex].payload
            }
          }
          break
        case "down":
          if (length > 0 && this.currentIndex > -1) {
            if (this.currentIndex === length - 1) {
              this.currentIndex = -1
              this.communication.input = ""
            } else if (this.currentIndex < length - 1) {
              this.currentIndex = this.currentIndex + 1
              this.communication.input =
                clientMessages[this.currentIndex].payload
            }
          }
          break
      }
    },
    addProtocol() {
      this.protocols.push({ value: "", active: true })
    },
    deleteProtocol({ index }) {
      const oldProtocols = this.protocols.slice()
      this.$delete(this.protocols, index)
      this.$toast.success(this.$t("state.deleted"), {
        icon: "delete",
        action: {
          text: this.$t("action.undo"),
          duration: 4000,
          onClick: (_, toastObject) => {
            this.protocols = oldProtocols
            toastObject.remove()
          },
        },
      })
    },
  },
})
</script>
