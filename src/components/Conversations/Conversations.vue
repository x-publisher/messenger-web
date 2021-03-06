<template>
    <div id="conversation-list" class="page-content">

        <!-- Spinner On load -->
        <spinner class="spinner" v-if="conversations.length == 0 && loading"></spinner>

        <div id="quick_find" v-if="showSearch">
           <div>
             <input v-model="searchQuery" id="search-bar" class="quick_find fixed_pos" type="text text_box" placeholder="Search conversations..." autocomplete="off" autocorrect="off" spellcheck="false">
           </div>
         </div>

        <!-- If no Messages -->
        <p class="empty-message" v-if="conversations.length == 0 && !loading">{{ $t('conversations.noconv') }}</p>

        <!-- Conversation items -->
        <transition-group name="flip-list" tag="div">
            <component v-for="conversation in conversations" :is="conversation.title ? 'ConversationItem' : 'DayLabel'" :conversation-data="conversation" :archive="isArchive" :small="small" :key="conversation.hash ? conversation.hash : conversation.label"/>
        </transition-group>

        <button tag="button" class="compose mdl-button mdl-js-button mdl-button--fab mdl-js-ripple-effect mdl-button--colored" @click="$router.push('/compose');" :style="composeStyle" v-if="!small" v-mdl>
            <i class="material-icons md-light">add</i>
        </button>
    </div>
</template>

<script>
import Vue from 'vue';
import { i18n } from '@/utils'
import Hash from 'object-hash'
import { Util, Api, SessionCache, TimeUtils } from '@/utils'
import ConversationItem from './ConversationItem.vue'
import DayLabel from './DayLabel.vue'
import Spinner from '@/components/Spinner.vue'
import emojione from 'emojione'

export default {
    name: 'conversations',
    props: ['small', 'index', 'folderId', 'folderName'],

    mounted () {
        this.$store.state.msgbus.$on('newMessage', this.updateConversation);
        this.$store.state.msgbus.$on('conversationRead', this.updateRead);
        this.$store.state.msgbus.$on('removedConversation', this.fetchConversations);
        this.$store.state.msgbus.$on('refresh-btn', this.refresh);
        this.$store.state.msgbus.$on('newMargin', this.updateMargin);
        this.$store.state.msgbus.$on('search-btn', this.toggleSearch);

        this.fetchConversations();

        if (!this.small) {
            // Construct colors object from saved global theme
            const colors = {
                'default': this.$store.state.theme_global_default,
                'dark': this.$store.state.theme_global_dark,
                'accent': this.$store.state.theme_global_accent,
            };

            // Commit them to current application colors
            this.$store.commit('colors', colors);
        }
    },

    beforeDestroy () {
        // when coming from a thread, back to the conversation list, this beforeDestory
        // was getting called after the mounted callback, which erased the bus functionality.
        // when it is mounted, it is overriding the old action, anyways.

        if (!this.small) {
            this.$store.state.msgbus.$off('newMessage')
            this.$store.state.msgbus.$off('conversationRead')
            this.$store.state.msgbus.$off('refresh-btn');
            this.$store.state.msgbus.$off('search-btn');
            this.$store.state.msgbus.$off('newMargin');
        }
    },

    methods: {

        fetchConversations () {
            if (this.index == "index_private") {
                let lastPasscodeEntry = this.$store.state.last_passcode_entry;

                // no recent passcode entry
                if (lastPasscodeEntry == null || lastPasscodeEntry < (Date.now() - (15 * 1000))) {
                    this.$router.push('/passcode');
                    return;
                }
            }

            // Start query
            Api.conversations.getList(this.index, this.folderId)
                .then(response => this.processConversations(response));
        },

        processConversations (response, updateUnfiltered = true) {
            if (updateUnfiltered) {
                // used for searching
                this.unFilteredAllConversations = response;
            }

            const updatedConversations = [];

            const cache = [];
            const titles = [];

            for(let i in response) {
                const item = response[i]
                if (typeof item == "function") {
                    continue;
                }

                const title = this.calculateTitle(item);

                if (titles.indexOf(title) == -1) {
                    titles.push(title);

                    updatedConversations.push({
                        label: title,
                        hash: Hash(title)
                    });
                }

                updatedConversations.push(item)

                // Save to contact cache
                cache.push(
                    Util.generateContact(
                        item.device_id,
                        item.title,
                        item.phone_numbers,
                        item.mute,
                        item.private_notifications,
                        item.color,
                        Util.expandColor(item.color_accent),
                        Util.expandColor(item.color_light),
                        Util.expandColor(item.color_dark)
                    )
                );

            }

            this.loading = false;
            this.$store.commit('conversations', cache);
            this.conversations = updatedConversations;

            if (!this.small) {
                this.$store.commit("loading", false);
                this.$store.commit('title', this.index == 'folder' ? this.folderName : this.title);
            }

        },

        updateConversation (event_obj) {

            if (this.searchClicked) {
                this.processConversations(this.unFilteredAllConversations);
                this.searchClicked = false;
            }

            // Find conversation
            let { conv, conv_index } = this.getConversation(event_obj.conversation_id);

            if (!conv || !conv_index) {
                // if the conversation doesn't exist, we have a problem, or it is a new conversation.
                // invalidate the refresh the list from the API.
                // It is better to fix the actual problem and update the messages correctly though, without the refresh.
                SessionCache.invalidateAllConversations();
                this.fetchConversations();

                return false;
            }

            // Generate new snippet
            let new_snippet = emojione.unicodeToImage(Util.generateSnippet(event_obj));

            conv.snippet = new_snippet;
            conv.read = event_obj.read;

            conv.hash = Hash(conv);

            // Get start index (index after pinned items)
            let startIndex = 0;
            if (this.conversations[0].label == "Pinned" && !conv.pinned) { // If there are any pinned items
                this.conversations.some( (conv, i) => {
                    if (typeof conv.label != "undefined" // Loop until we find a label
                        && conv.label != "Pinned") { // That is not "pinned"

                        startIndex = i; // Save index and return
                        return true
                    }
                })
            }

            // Move conversation if required
            if (conv_index != startIndex + 1) {
                conv = this.conversations.splice(conv_index, 1)[0]

                // If top label is not "Today"
                // This isn't elegant, but it works
                if (this.conversations[startIndex].label != "Today"
                    && this.conversations[startIndex].label != "Pinned") {
                    const title = "Today"; // Define title
                    const label = {        // And Define Label
                        label: title,
                        hash: Hash(title)
                    }

                    // Push label and conversation
                    this.conversations.splice(startIndex, 0, label, conv)
                } else { // Else, just push the converstation to index 1 (below label)
                    this.conversations.splice(startIndex + 1, 0, conv)
                }
            }

            this.conversations = this.conversations;
        },

        updateRead (id) {

            let { conv, conv_index } = this.getConversation(id);

            if(!conv || !conv_index)
                return false;

            conv.read = true;
            conv.hash = Hash(conv)
        },

        getConversation(id) {

            let conv_index = null;
            let conv = null;

            for(conv_index in this.conversations) {
                conv = this.conversations[conv_index];

                if(id == conv.device_id)
                    return  { conv, conv_index };
            }

            return  { conv: null, conv_index: -1 };
        },

        /**
         * refresh
         * Force refresh messages - fetches from server
         */
        refresh () {
            //if (!this.small) // Don't clear list if using sidebar list
                //this.conversations = [];

            this.loading = true;
            SessionCache.invalidateAllConversations();
            this.fetchConversations();
        },

        updateMargin (margin) {
            this.margin = margin;
        },

        toggleSearch () {
            this.searchClicked = !this.searchClicked;

            if (this.searchClicked) {
                Vue.nextTick(() => { // Wait item to render
                    this.$el.querySelector('#search-bar').focus();
                });
            } else {
                this.searchQuery = "";
            }
        },

        calculateTitle (conversation) {
            if (conversation.pinned)
                return i18n.t('conversations.pinned');
            else if (TimeUtils.isToday(conversation.timestamp))
                return i18n.t('conversations.today');
            else if (TimeUtils.isYesterday(conversation.timestamp))
                return i18n.t('conversations.yesterday');
            else if (TimeUtils.isLastWeek(conversation.timestamp))
                return i18n.t('conversations.thisweek');
            else if (TimeUtils.isLastMonth(conversation.timestamp))
                return i18n.t('conversations.thismonth');
            else
                return i18n.t('conversations.older');
        }
    },

    data () {
        return {
            title: "",
            loading: true,
            conversations: [],
            unFilteredAllConversations: [],
            margin: 0,
            searchClicked: false,
            searchQuery: ""
        }
    },

    computed: {
        isArchive () {
            return this.index == "index_archived";
        },

        composeStyle () {
            return "background: " + this.$store.state.colors_accent + "; " +
                    "marginRight: " + (this.margin + 36) + "px;";
        },

        showSearch() {
            return this.searchClicked && !this.small;
        }
    },

    watch: {
        '$route' (to, from) { // Update index on route change

            // Only update if list page
            if (to.name != from.name && to.name.indexOf('conversations-list') >= 0) {
                this.conversations = [];
                this.unFilteredAllConversations = [];

                this.fetchConversations();
            }

        },

        "searchQuery" (to, from) {
            to = to.toLowerCase();
            let filteredConversations = [];

            for (let i in this.unFilteredAllConversations) {
                let conversation = this.unFilteredAllConversations[i];

                if (typeof conversation == "function") {
                    continue;
                }

                if (conversation.title.toLowerCase().indexOf(to) > -1 ||
                        conversation.snippet.toLowerCase().indexOf(to) > -1 ||
                        conversation.phone_numbers.indexOf(to) > -1) {
                    filteredConversations.push(conversation);
                }
            }

            this.processConversations(filteredConversations, false);
        }
    },

    components: {
        ConversationItem,
        DayLabel,
        Spinner
    }
}
</script>

<!-- Add "scoped" attribute to limit CSS to this component only -->
<style lang="scss" scoped>
    @import "../../assets/scss/_vars.scss";

    .empty-message {
        color: rgba(0, 0, 0, 0.54);
        margin: 6em auto;
        width: 9.5em;
    }

    .compose {
        position: fixed;
        bottom: 0%;
        right: 0%;
        z-index: 3;
        margin: 24px;
    }

    #conversation-list {
        width: 100%;
        margin-top: 36px !important;

        .spinner {
            margin-top: 100px;
        }
    }

    #quick_find {
      white-space: nowrap;
      padding-top: 5px;
      text-align: right;
    }

    .quick_find {
      width: 215px;
      margin-top: 3px;
      border: 0px solid white;
      border-radius: 2px;
      font-size: 15px;
      background-color: white;
      color: black;
      background-position: 10px 10px;
      background-repeat: no-repeat;
      padding: 12px 16px 12px 16px;
      -webkit-transition: width 0.4s ease-in-out;
      transition: width 0.4s ease-in-out;
      box-shadow: 0px 2px 2px rgba(0, 0, 0, .3);
    }

    .quick_find:focus {
      width: 400px;
      outline: none !important;
    }

    @media (max-width:450px) {
      .quick_find:focus {
        width: 250px;
      }
    }

    .flip-list-enter, .flip-list-leave-to	{
        opacity: 0;
    }

    .flip-list-leave-active {
        position: absolute;
    }

    .flip-list-move {
        transition: transform $anim-time;
    }

    body.dark {
        .empty-message {
            color: rgba(255, 255, 255, 0.54);
        }

        .quick_find {
          border: 0px solid $bg-darker;
          background-color: $bg-darker;
          color: white;
        }
    }
</style>
