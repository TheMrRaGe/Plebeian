<!--
//  ChatWindow.vue
//
//  Created by Kalila L. on May 13th, 2021.
//  Copyright 2021 Vircadia contributors.
//  Copyright 2022 DigiSomni LLC.
//
//  Distributed under the Apache License, Version 2.0.
//  See the accompanying file LICENSE or http://www.apache.org/licenses/LICENSE-2.0.html
-->

<style lang="scss" scoped>
    .q-field {
        background-color: rgba(0, 0, 0, 0.4);
    }
</style>

<style lang="scss">
    .q-message-text-content a {
        color: white;
    }
</style>

<template>
    <OverlayShell
        icon="chat"
        title="Chat"
        :managerProps="propsToPass"
        :defaultHeight="300"
        :defaultWidth="600"
        :defaultLeft="0"
        :defaultTop="400"
        spawnSnappedTo="bottom"
        :hoverShowBar="true"
        :transparentOnIdle="true"
        :style="{
            'box-shadow': '0 1px 5px rgb(0 0 0 / 20%), 0 2px 2px rgb(0 0 0 / 14%), 0 3px 1px -2px rgb(0 0 0 / 12%)',
            border: 'none'
        }"
    >
        <q-card
            class="column no-wrap items-stretch full-height"
            style="background: transparent; box-shadow: none;"
        >
            <q-scroll-area
                ref="listOfChats"
                class="col"
                style="height: 100%"
                @scroll="scrollToLatestMessage"
            >
                <q-card-section class="q-pt-none">
                    <template v-for="(msg, index) in sortedMessages" :key="index">
                        <q-chat-message
                            :text="msg.text"
                            :text-html="true"
                            :sent="msgIsFromThisClient(msg.root.senderId)"
                            :name="
                                msgIsFromThisClient(msg.root.senderId) ? `${msgSender(msg.root)} (you)` : msgSender(msg.root)
                            "
                            :stamp="formatMessageTime(msg.time)"
                            text-color="white"
                            :bg-color="msgIsFromThisClient(msg.root.senderId) ? 'primary' : 'grey-9'"
                        >
                            <template v-slot:avatar>
                                <q-avatar
                                    :color="msgIsFromThisClient(msg.root.senderId) ? 'primary' : 'grey-9'"
                                    class="q-mx-xs"
                                >
                                    <img
                                        v-if="getProfilePicture(msgSender(msg.root))"
                                        :src="getProfilePicture(msgSender(msg.root))"
                                    >
                                    <span v-else>{{ msgSender(msg.root).charAt(0) }}</span>
                                </q-avatar>
                            </template>
                        </q-chat-message>
                    </template>
                </q-card-section>
            </q-scroll-area>
            <q-input
                class="inputBox"
                placeholder="Type a message..."
                outlined
                v-model="messageInput"
                :dense="true"
                @keydown.enter.prevent="submitMessage"
            >
                <template v-slot:append>
                    <q-btn
                        title="Click, or press Enter to send."
                        flat
                        round
                        dense
                        @click.stop="submitMessage"
                    >
                        <q-icon name="send" color="primary"></q-icon>
                    </q-btn>
                </template>
            </q-input>
        </q-card>
        <!-- <q-inner-loading :showing="">
            <q-spinner-gears size="50px" color="primary" />
        </q-inner-loading> -->
    </OverlayShell>
</template>

<script lang="ts">
import { defineComponent } from "vue";
import * as DOMPurify from "dompurify";
import OverlayShell from "../OverlayShell.vue";

import { AMessage, DomainMessage, DefaultChatMessage } from "@Modules/domain/message";
import { DomainMgr } from "@Modules/domain";
import { Uuid } from "@vircadia/web-sdk";

// Interface for Quasar-compatible messages that have had their text fields combined.
interface ACombinedMessage {
    root: AMessage,
    text: string[],
    time: Date
}

export default defineComponent({
    name: "ChatWindow",

    props: {
        propsToPass: { type: Object, default: () => ({}) }
    },

    components: {
        OverlayShell
    },

    data: () => ({
        messageInput: "",
        // The reason for using "self" instead of checking if the username matches our own
        // is because a user may write chat messages as a guest. Checking "displayName"
        // won"t help much either as anyone can choose any display name and cause confusion.
        subscribed: false,  // 'true' if subscribed for messages

        // Following is legacy test data. Can be removed when chat is working
        worldChatHistory: [
            {
                displayName: "Hallo",
                username: "nani",
                self: false,
                timestamp: new Date().toString(),
                message: "Hi, hru?"
            },
            {
                displayName: "Waifu",
                username: "testerino",
                self: true,
                timestamp: new Date().toString(),
                message: "Sup holmes."
            },
            {
                displayName: "Hallo",
                username: "nani",
                self: false,
                timestamp: new Date().toString(),
                message: "nammuch you?"
            },
            {
                displayName: "Waifu",
                username: "testerino",
                self: true,
                timestamp: new Date().toString(),
                message: "you know the life."
            }
        ],

        currentPrimaryMessages: [],
        previousScrollPos: 0,
        scrollIsAtBottom: true,
        lastPrimaryMessageIndex: -1,

        messageCombinationTimeLimit: 120000, // 2 minutes.
        sortedMessages: [] as ACombinedMessage[]
    }),

    computed: {
    },

    watch: {
        /*
        // When the message input changes, send the message
        messageInput: function(val: string): void {
            if (DomainMgr.ActiveDomain) {
                const msger = DomainMgr.ActiveDomain.MessageClient;
                if (msger) {
                    const msg: DefaultChatMessage = {
                        type: "TransmitChatMessage",
                        channel: DomainMessage.DefaultChatChannel,
                        message: val,
                        colour: { red: 255, blue: 255, green: 255 },
                        displayName: this.$store.state.avatar.displayName,
                        position: this.$store.state.avatar.position
                    };
                    msger.sendMessage(DomainMessage.DefaultChatChannel, JSON.stringify(msg));
                }
            }
        }
        */
    },

    methods: {
        sortMessages(messages: AMessage[]): ACombinedMessage[] {
            const liveMessages = messages;
            // Skip sorting if there are no messages.
            if (liveMessages.length <= 0) {
                return [];
            }

            // Sort the messages.
            const sortedMessages = [] as ACombinedMessage[];
            let merges = 0;
            liveMessages.forEach((message, index) => {
                // Get the previous message in the chat.
                // Account for the number of already combined messages.

                const previousMessage = sortedMessages[index - 1 - merges];
                // If this is the first message to be sorted, simply append it to the list.
                if (sortedMessages.length <= 0 || !previousMessage) {
                    sortedMessages.push({
                        root: message,
                        text: [this.sanitizeMessageText(this.formatMessageLinks(this.msgText(message)))],
                        time: message.whenReceived
                    });
                    // Move on to the next message.
                    return;
                }

                // If the sender ID is the same as the previous message,
                // and this message is within the time window of the previous message.
                if (
                    previousMessage.root.senderId.stringify() === message.senderId.stringify()
                    && previousMessage.time.getTime() < message.whenReceived.getTime()
                    && message.whenReceived.getTime() < previousMessage.time.getTime()
                    + this.messageCombinationTimeLimit
                ) {
                    // Merge this message into the previous one.
                    previousMessage.text.push(this.sanitizeMessageText(this.formatMessageLinks(this.msgText(message))));
                    previousMessage.time = message.whenReceived;
                    merges += 1;
                } else {
                    // Otherwise, simply append this message to the list.
                    sortedMessages.push({
                        root: message,
                        text: [this.sanitizeMessageText(this.formatMessageLinks(this.msgText(message)))],
                        time: message.whenReceived
                    });
                }
            });
            return sortedMessages;
        },
        getProfilePicture(username: string): string | undefined {
            // Should store profile pictures after retrieving and then pull each
            // subsequent one from cache instead of hitting metaverse every time.

            // This is filler functionality to enable the UI to be developed more correctly now.
            if (username === "testerino") {
                return "https://cdn.quasar.dev/img/avatar4.jpg";
            }
            return undefined;
        },
        // Return the sender Id included in the message. Returns the ID string if no displayname
        msgSender(pMsg: AMessage): string {
            if (pMsg.messageJSON) {
                // eslint-disable-next-line max-len
                // eslint-disable-next-line @typescript-eslint/no-unsafe-assignment,@typescript-eslint/no-unnecessary-type-assertion
                const fMsg = <DefaultChatMessage>pMsg.messageJSON;
                // eslint-disable-next-line @typescript-eslint/no-unsafe-return,@typescript-eslint/no-unsafe-member-access
                return fMsg.displayName;
            }
            return pMsg.senderId.stringify();
        },
        // Return the text of a messsage. If DefaultMessage, use the text in the JSON packet.
        msgText(message: AMessage): string {
            if (message.messageJSON) {
                const dMsg = message.messageJSON as DefaultChatMessage;
                if ("message" in dMsg) {
                    return dMsg.message;
                }
            }
            return message.message;
        },
        formatMessageLinks(text: string): string {
            // Find all URLs in the message.
            // eslint-disable-next-line require-unicode-regexp
            const regex = /\bhttps?:\/\/\S+|\bwss?:\/\/\S+/gi;
            // Wrap each URL in an anchor tag.
            const matches = text.replace(
                regex,
                (match) => `<a href="${match}" target="_blank" rel="noreferrer">${match}</a>`
            );
            return matches;
        },
        // Sanitize message content to reduce the risk of XSS attacks.
        sanitizeMessageText(text: string): string {
            return DOMPurify.sanitize(text, { USE_PROFILES: { html: true } });
        },
        // Return the printable time of the message.
        formatMessageTime(time: Date): string {
            // Messages loaded from persistent storage can contain broken timestamps.
            if (!time) {
                return "";
            }
            // Show in local time.
            return `${time.toDateString()}, ${time.toLocaleTimeString()}`;
        },
        // Return 'true' if the chat message was sent by this session
        // Relies of the message queuer to add the "self: true" to the added message
        msgSentBySelf(pMsg: AMessage): boolean {
            // eslint-disable-next-line @typescript-eslint/no-unsafe-return
            return pMsg.self ?? false;
        },
        msgIsFromThisClient(senderId: Uuid | string): boolean {
            if (DomainMgr.ActiveDomain) {
                const ID = DomainMgr.ActiveDomain.DomainClient?.sessionUUID;
                if (
                    senderId instanceof Uuid && ID?.value() === senderId?.value()
                    || typeof senderId === "string" && ID?.value().toString() === senderId
                ) {
                    return true;
                }
            }
            return false;
        },
        validateMessage(): boolean {
            return this.messageInput.length > 0;
        },
        submitMessage(): void {
            if (DomainMgr.ActiveDomain) {
                const msger = DomainMgr.ActiveDomain.MessageClient;
                if (msger && this.validateMessage()) {
                    const msg: DefaultChatMessage = {
                        type: "TransmitChatMessage",
                        channel: "Local",
                        message: this.sanitizeMessageText(this.messageInput),
                        colour: { red: 255, blue: 204, green: 229 }, // orangish
                        displayName: this.$store.state.avatar.displayName,
                        position: this.$store.state.avatar.position
                    };
                    msger.sendMessage(DomainMessage.DefaultChatChannel, JSON.stringify(msg));
                    // Clear the input field.
                    this.messageInput = "";
                    // Scroll to the bottom of the chat.
                    this.scrollToBottom(false);
                }
            }
        },
        scrollToLatestMessage({ verticalPosition }: { verticalPosition: number }, smooth = true): void {
            const listOfChats = this.$refs.listOfChats as { $el: HTMLElement };
            const scrollElem = listOfChats.$el.querySelector(".scroll") as HTMLElement;
            const scrollHeight = scrollElem.scrollHeight;
            const scrollMargin = 20;
            // A new message will trigger the scroll event without changing the scroll position.
            const newMessage = verticalPosition === this.previousScrollPos;
            if (newMessage) {
                if (newMessage && this.scrollIsAtBottom) {
                    scrollElem.scrollTo({
                        top: scrollHeight,
                        left: 0,
                        behavior: smooth ? "smooth" : "auto"
                    });
                }
            } else if (verticalPosition < scrollHeight - scrollElem.clientHeight - scrollMargin) {
                this.scrollIsAtBottom = false;
            } else {
                this.scrollIsAtBottom = true;
            }
            this.previousScrollPos = verticalPosition;
        },
        scrollToBottom(smooth = true): void {
            const listOfChats = this.$refs.listOfChats as { $el: HTMLElement };
            const scrollElem = listOfChats.$el.querySelector(".scroll") as HTMLElement;
            const scrollHeight = scrollElem.scrollHeight;
            scrollElem.scrollTo({
                top: scrollHeight,
                left: 0,
                behavior: smooth ? "smooth" : "auto"
            });
            this.scrollIsAtBottom = true;
        },
        relatedToPrimaryMessageEntry(pMsg: AMessage, index: number): boolean {
            const lastPrimaryMessage = this.$store.state.messages.messages[this.lastPrimaryMessageIndex];

            if (!lastPrimaryMessage) {
                this.lastPrimaryMessageIndex = index;
                return false;
            }

            if (pMsg.senderId !== lastPrimaryMessage.senderId) {
                this.lastPrimaryMessageIndex = index;
                return false;
            }

            // eslint-disable-next-line @typescript-eslint/no-magic-numbers
            if (Math.abs(lastPrimaryMessage.whenReceived.getTime() - pMsg.whenReceived.getTime()) > 5000) {
                this.lastPrimaryMessageIndex = index;
                return false;
            }

            return true;
        }
    },

    mounted() {
        // Sort existing messages.
        this.sortedMessages = this.sortMessages(this.$store.state.messages.messages);

        // Sort any new messages.
        this.$store.watch(
            (state) => JSON.stringify(
                state.messages.messages,
                (key: string, entry: unknown) => {
                    // Convert BigInt values to strings, since there is no default serializer for them.
                    if (entry instanceof window.BigInt) {
                        return entry.toString();
                    }
                    return entry;
                }
            ),
            () => {
                this.sortedMessages = this.sortMessages(this.$store.state.messages.messages);
            }
        );

        // Scroll to the bottom of the chat.
        this.scrollToBottom(false);
    }
});
</script>
