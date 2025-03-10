<script lang="ts">
    import { onMount } from 'svelte';
    import { now } from 'svelte/internal';

    import Modal from './Modal.svelte';

    import Content from './Content.svelte';
    import Message from './Message.svelte';
    import SSOButton from './SSOButton.svelte';
    import LogoutButton from './LogoutButton.svelte';
    import SSORevokeButton from './SSORevokeButton.svelte';
    import { validate } from './session';

    import { isAuthenticated } from './store';
    // subscribe to the authentication status
    let isLoggedIn: boolean;
    isAuthenticated.subscribe((status) => {
        isLoggedIn = status;
    });

    export let messages: messageType[] = [];

    let noOfMessages: number;
    let inputMessages: number;
    let content: messageType;
    let filter: string;
    let isDevOn: boolean;
    let conn: WebSocket;
    let follow: boolean = true;
    let showModal;

    // truncate the number of messages show in the app
    $: if (messages.length > noOfMessages) {
        messages = messages.slice(1);
    }

    $: if (isDevOn) {
        test();
    } else {
        dial(conn);
    }

    let filterPorts: number[] = [];

    function defineMessages(event: any) {
        let chosenValue = event.target[0].value;
        let presentMessages = messages.length;

        if (chosenValue <= 0) {
            return;
        }

        if (chosenValue < presentMessages) {
            messages = messages.slice(messages.length - chosenValue, messages.length);
        }

        noOfMessages = chosenValue;
    }

    function toggleMode(event: any) {
        let chosenMode = event.target.id;

        if (chosenMode == 'dev') {
            isDevOn = true;
        } else if (chosenMode == 'prod') {
            isDevOn = false;
            if (conn != null) {
                conn.close();
            }
        }
    }

    function addMessage(message: messageType) {
        if (message.dstPort === null || !filterPorts.includes(message.dstPort)) {
            messages.push(message);
            messages = messages;
        }
    }

    function filterMessages() {
        filterPorts = filter.split('&&').map(Number);
        messages = messages.filter((message) => !filterPorts.includes(message.dstPort));
    }

    function dial(conn: WebSocket) {
        let wsUrl =
            location.protocol === 'https:'
                ? `wss://${location.host}/subscribe`
                : `ws://${location.host}/subscribe`;
        conn = new WebSocket(wsUrl);

        if (conn) {
            conn.addEventListener('close', (ev) => {
                if (ev.code !== 1001) {
                    //appendLog("Reconnecting in 1s");
                    setTimeout(dial, 1000);
                }
            });
            conn.addEventListener('open', () => {
                console.info('websocket connected');
            });
            conn.addEventListener('message', (ev) => {
                const obj = JSON.parse(ev.data);
                console.log(obj);
                addMessage(obj);
            });
        }
        return true;
    }

    function displayContent(event: any) {
        content = event.detail;
    }

    const sleep = (ms: number) => new Promise((f) => setTimeout(f, ms));

    const test = async () => {
        while (isDevOn) {
            await sleep(1000);
            addMessage({
                action: 'action',
                connKey: [2, 2],
                dstPort: 1234,
                rule: 'Rule: TCP',
                scanner: 'censys',
                sensorID: 'sensorID',
                srcHost: '1.1.1.1',
                srcPort: '4321',
                timestamp: now().toString(),
                payload: 'dGVzdA==', // test
            });
        }
    };

    onMount(() => {
        // Default value of number of messages
        noOfMessages = 50;
        isDevOn = false;
        conn = null;
        validate();
    });
</script>

<Modal bind:showModal>
    <form id="configmodal" on:submit|preventDefault={defineMessages}>
        <p>Number of messages</p>
        <input
            id="messages-input-box"
            type="number"
            min="0"
            bind:value={inputMessages}
            class:error-state={inputMessages <= 0}
        />
        <p>Mode</p>
        <label>
            <input type="radio" name="radio-group" id="dev" on:click={toggleMode} />
            Development
        </label>
        <label>
            <input type="radio" name="radio-group" checked id="prod" on:click={toggleMode} />
            Production
        </label>
        <button disabled={inputMessages < 0} type="submit">Apply</button>
    </form>
</Modal>

<header class="site-header">
    <b>Ochi</b>: find me at
    <a target="_blank" href="https://github.com/glaslos/ochi">github/glaslos/ochi</a>
    <input bind:value={filter} placeholder="Filter destination port" />
    <button on:click={filterMessages}>Apply</button>
    <span>Port number and '&&' to concat.</span>
    <button
        id="configButton"
        on:click={() => {
            showModal = true;
            inputMessages = noOfMessages;
        }}>Config</button
    >
    {#if !isLoggedIn}
        <SSOButton />
    {:else}
        <LogoutButton />
        <SSORevokeButton />
    {/if}
</header>

<main>
    <div class="row">
        <div
            class="column"
            id="message-log"
            on:wheel={() => {
                follow = false;
            }}
        >
            {#each messages as message (message.timestamp)}
                {#if !filterPorts.includes(message.dstPort)}
                    <Message on:message={displayContent} {message} {follow} />
                {/if}
            {/each}
        </div>
        <Content {content} />
    </div>

    {#if !follow}
        <button
            on:click={() => {
                follow = true;
            }}
            id="resume-btn">Resume</button
        >
    {/if}
</main>

<style>
    .site-header {
        border-bottom-style: solid;
        border-width: 1px;
    }

    main {
        width: 100vw;
        min-width: 320px;
    }

    .row {
        display: flex;
        position: absolute;
        top: 55px;
        left: 0;
        bottom: 0;
        right: 0;
    }

    .column {
        flex: 50%;
        padding: 15px 20px;
    }

    #message-log {
        width: 100%;
        flex-grow: 1;
        overflow-y: scroll;
    }

    #resume-btn {
        position: fixed;
        bottom: 2rem;
        z-index: 2;
        left: 40vw;
        cursor: pointer;
    }

    .error-state {
        border: 2px red solid;
        outline: none;
    }
</style>
