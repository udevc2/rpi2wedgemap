<template>

    <MessagesSidebar
        :conversations="conversations"
        :peers="peers"
        :selected-destination-hash="selectedPeer?.destination_hash"
        @conversation-click="onConversationClick"
        @peer-click="onPeerClick"/>

    <div class="flex flex-col flex-1 overflow-hidden min-w-full sm:min-w-[500px] dark:bg-zinc-950">

        <!-- messages tab -->
        <ConversationViewer
            ref="conversation-viewer"
            :my-lxmf-address-hash="config?.lxmf_address_hash"
            :selected-peer="selectedPeer"
            :conversations="conversations"
            @close="onCloseConversationViewer"
            @reload-conversations="getConversations"/>

    </div>

</template>

<script>

import WebSocketConnection from "../../js/WebSocketConnection";
import MessagesSidebar from "./MessagesSidebar.vue";
import ConversationViewer from "./ConversationViewer.vue";
import Utils from "../../js/Utils";
import GlobalState from "../../js/GlobalState";
import DialogUtils from "../../js/DialogUtils";
import GlobalEmitter from "../../js/GlobalEmitter";

export default {
    name: 'MessagesPage',
    components: {
        ConversationViewer,
        MessagesSidebar,
    },
    props: {
        destinationHash: String,
    },
    data() {
        return {

            reloadInterval: null,

            config: null,
            peers: {},
            selectedPeer: null,

            conversations: [],
            lxmfDeliveryAnnounces: [],

        };
    },
    beforeUnmount() {

        clearInterval(this.reloadInterval);

        // stop listening for websocket messages
        WebSocketConnection.off("message", this.onWebsocketMessage);
        GlobalEmitter.off("compose-new-message", this.onComposeNewMessage);

    },
    mounted() {

        // listen for websocket messages
        WebSocketConnection.on("message", this.onWebsocketMessage);
        GlobalEmitter.on("compose-new-message", this.onComposeNewMessage);

        this.getConfig();
        this.getConversations();
        this.getLxmfDeliveryAnnounces();

        // update info every few seconds
        this.reloadInterval = setInterval(() => {
            this.getConversations();
        }, 5000);

        // compose message if a destination hash was provided on page load
        if(this.destinationHash){
            this.onComposeNewMessage(this.destinationHash);
        }

    },
    methods: {
        async onComposeNewMessage(destinationHash) {

            // ask for destination address if not provided
            if(destinationHash == null){
                destinationHash = await DialogUtils.prompt("Enter LXMF Address");
                if(!destinationHash){
                    return;
                }
            }

            // if user provided an address with an "lxmf@" prefix, lets remove that to get the raw destination hash
            if(destinationHash.startsWith("lxmf@")){
                destinationHash = destinationHash.replace("lxmf@", "");
            }

            // fetch updated announce as we might be composing new message before we loaded the announces list
            await this.getLxmfDeliveryAnnounce(destinationHash);

            // attempt to find existing peer so we can show their name
            const existingPeer = this.peers[destinationHash];
            if(existingPeer){
                this.onPeerClick(existingPeer);
                return;
            }

            // simple attempt to prevent garbage input
            if(destinationHash.length !== 32){
                DialogUtils.alert("Invalid Address");
                return;
            }

            // we didn't find an existing peer, so just use an unknown name
            this.onPeerClick({
                display_name: "Unknown Peer",
                destination_hash: destinationHash,
            });

        },
        async getConfig() {
            try {
                const response = await window.axios.get(`/api/v1/config`);
                this.config = response.data.config;
            } catch(e) {
                // do nothing if failed to load config
                console.log(e);
            }
        },
        async onWebsocketMessage(message) {
            const json = JSON.parse(message.data);
            switch(json.type){
                case 'config': {
                    this.config = json.config;
                    break;
                }
                case 'announce': {
                    const aspect = json.announce.aspect;
                    if(aspect === "lxmf.delivery"){
                        this.updatePeerFromAnnounce(json.announce);
                    }
                    break;
                }
                case 'lxmf.delivery': {
                    // reload conversations when a new message is received
                    await this.getConversations();
                    break;
                }
            }
        },
        async getLxmfDeliveryAnnounces() {
            try {

                // fetch announces for "lxmf.delivery" aspect
                const response = await window.axios.get(`/api/v1/announces`, {
                    params: {
                        aspect: "lxmf.delivery",
                        limit: 500, // limit ui to showing 500 latest announces
                    },
                });

                // update ui
                const lxmfDeliveryAnnounces = response.data.announces;
                for(const lxmfDeliveryAnnounce of lxmfDeliveryAnnounces){
                    this.updatePeerFromAnnounce(lxmfDeliveryAnnounce);
                }

            } catch(e) {
                // do nothing if failed to load announces
                console.log(e);
            }
        },
        async getLxmfDeliveryAnnounce(destinationHash) {
            try {

                // fetch announce for destination hash
                const response = await window.axios.get(`/api/v1/announces`, {
                    params: {
                        destination_hash: destinationHash,
                        limit: 1,
                    },
                });

                // update ui
                const lxmfDeliveryAnnounces = response.data.announces;
                for(const lxmfDeliveryAnnounce of lxmfDeliveryAnnounces){
                    this.updatePeerFromAnnounce(lxmfDeliveryAnnounce);
                }

            } catch(e) {
                // do nothing if failed to load announce
                console.log(e);
            }
        },
        async getConversations() {
            try {
                const response = await window.axios.get(`/api/v1/lxmf/conversations`);
                this.conversations = response.data.conversations;
            } catch(e) {
                // do nothing if failed to load conversations
                console.log(e);
            }
        },
        updatePeerFromAnnounce: function(announce) {
            this.peers[announce.destination_hash] = announce;
        },
        onPeerClick: function(peer) {

            // update selected peer
            this.selectedPeer = peer;

            // update current route
            this.$router.replace({
                name: "messages",
                params: {
                    destinationHash: peer.destination_hash,
                },
            });

        },
        onConversationClick: function(conversation) {

            // object must stay compatible with format of peers
            this.onPeerClick(conversation);

            // mark conversation as read
            this.$refs["conversation-viewer"].markConversationAsRead(conversation);

        },
        onCloseConversationViewer: function() {

            // clear selected peer
            this.selectedPeer = null;

            // update current route
            this.$router.replace({
                name: "messages",
            });

        },
    },
    watch: {
        conversations() {

            // update global state
            GlobalState.unreadConversationsCount = this.conversations.filter((conversation) => {
                return conversation.is_unread;
            }).length;

        },
    },
}
</script>
