<script lang="ts">
    import { chatInputFocusStore } from "../../Stores/ChatStore";

    let searchActive = false;
    import { chatSearchBarValue, navChat, joignableRoom } from "../Stores/ChatStore";
    import LoadingSmall from "../images/loading-small.svelte";
    import LL from "../../../i18n/i18n-svelte";
    import { gameManager } from "../../Phaser/Game/GameManager";
    import { UserProviderMerger } from "../UserProviderMerger/UserProviderMerger";
    import OnlineUsersCount from "./OnlineUsersCount.svelte";
    import { IconMessageCircle2, IconSearch, IconUsers, IconX } from "@wa-icons";
    const gameScene = gameManager.getCurrentGameScene();
    const chat = gameManager.chatConnection;
    const showNavBar = gameScene.room.isChatOnlineListEnabled || gameScene.room.isChatDisconnectedListEnabled;
    const userProviderMergerPromise = gameScene.userProviderMerger;
    let searchLoader = false;
    const chatStatusStore = chat.connectionStatus;
    let typingTimer: ReturnType<typeof setTimeout>;
    const DONE_TYPING_INTERVAL = 2000;

    let searchValue = "";

    const handleKeyDown = () => {
        if (searchValue === "") joignableRoom.set([]);
        clearTimeout(typingTimer);
    };

    const handleKeyUp = (userProviderMerger: UserProviderMerger) => {
        clearTimeout(typingTimer);
        typingTimer = setTimeout(() => {
            searchLoader = true;
            if ($navChat === "chat" && $chatSearchBarValue.trim() !== "") {
                searchAccessibleRooms();
            }

            userProviderMerger.setFilter($chatSearchBarValue).finally(() => {
                searchLoader = false;
            });
        }, DONE_TYPING_INTERVAL);
    };

    const searchAccessibleRooms = () => {
        chat.searchAccessibleRooms($chatSearchBarValue)
            .then((chatRooms: { id: string; name: string | undefined }[]) => {
                joignableRoom.set(chatRooms);
            })
            .finally(() => {
                searchLoader = false;
            });
        return;
    };

    function focusChatInput() {
        // Disable input manager to prevent the game from receiving the input
        chatInputFocusStore.set(true);
    }
    function unfocusChatInput() {
        // Enable input manager to allow the game to receive the input
        chatInputFocusStore.set(false);
    }
</script>

<div class="tw-p-2 tw-flex tw-items-center tw-absolute tw-w-full tw-z-40">
    <div class={searchActive ? "tw-hidden" : ""}>
        {#if showNavBar}
            {#if $navChat === "chat"}
                <button
                    class="userList tw-p-3 hover:tw-bg-white/10 tw-rounded-xl tw-aspect-square tw-w-12"
                    on:click={() => navChat.switchToUserList()}
                >
                    <IconUsers font-size="20" />
                </button>
            {:else}
                <button
                    class="tw-p-3 hover:tw-bg-white/10 tw-rounded-2xl tw-aspect-square tw-w-12"
                    on:click={() => navChat.switchToChat()}
                >
                    <IconMessageCircle2 font-size="20" />
                </button>
            {/if}
        {/if}
    </div>
    <div class="tw-flex tw-flex-col tw-items-center tw-justify-center tw-grow">
        <div class="tw-text-md tw-font-bold tw-h-5 {searchActive ? 'tw-hidden' : ''}">
            {#if $navChat === "chat"}
                {$LL.chat.chat()}
            {:else}
                {$LL.chat.users()}
            {/if}
        </div>
        {#if gameScene.room.isChatOnlineListEnabled}
            <OnlineUsersCount {searchActive} />
        {/if}
    </div>
    <div class="">
        {#if $chatStatusStore !== "OFFLINE"}
            <button
                class="tw-p-3 hover:tw-bg-white/10 tw-rounded-xl tw-aspect-square tw-w-12 tw-relative tw-z-50"
                on:click={() => (searchActive = !searchActive)}
            >
                {#if searchLoader}
                    <LoadingSmall />
                {/if}
                {#if !searchActive}
                    <IconSearch font-size="20" />
                {:else}
                    <IconX font-size="20" />
                {/if}
            </button>
        {:else}
            <div class="tw-w-12" />
        {/if}
        <!-- searchbar -->
        {#if searchActive && $chatStatusStore !== "OFFLINE"}
            {#await userProviderMergerPromise}
                <div />
            {:then userProviderMerger}
                <div class="tw-absolute tw-w-full tw-h-full tw-z-40 tw-right-0 tw-top-0 tw-bg-contrast/30">
                    <input
                        autocomplete="new-password"
                        class="wa-searchbar tw-block tw-text-white placeholder:tw-text-white/50 tw-w-full placeholder:tw-text-sm tw-border-none tw-pl-6 tw-pr-20 tw-bg-transparent tw-py-3 tw-text-base tw-h-full"
                        placeholder={$navChat === "users" ? $LL.chat.searchUser() : $LL.chat.searchChat()}
                        on:keydown={handleKeyDown}
                        on:keyup={() => handleKeyUp(userProviderMerger)}
                        bind:value={$chatSearchBarValue}
                        on:focusin={focusChatInput}
                        on:focusout={unfocusChatInput}
                    />
                </div>
            {/await}
        {/if}
    </div>
</div>
