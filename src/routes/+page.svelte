<script>
    // deps
    import { onMount } from "svelte";
    import { marked } from "marked";
    import { dev } from "$app/environment";
    import SvelteMarkdown from "svelte-markdown";
    import katex from "$lib/katex.svelte";

    // state
    let state = null;
    let url = dev ? "http://localhost:8080/" : "/";
    let messages = [];
    let files = [];
    let filePicker;
    let textarea;
    let socket;
    let container;
    let chunks = [];
    let scrolled = true;
    let mediaRecorder;
    let sendingMessage = false;
    let recordingAudio = false;
    let streamingStarted = false;

    // send message
    async function sendMessage() {
        // set the message
        let message = textarea.value.trim();
        // return if the message is blank
        if (message.length === 0) return;
        // if we got an image, save it
        let image;
        let form;
        if (files.length !== 0) {
            image = files[0];
            form = new FormData();
            form.append("image", image);
            form.append("input", message);
            files = [];
        }
        // clear the textarea
        textarea.value = "";
        // change the textarea height
        textarea.style.height = "auto";
        textarea.style.height = `${textarea.scrollHeight + 4}px`;
        // set sending message to true
        sendingMessage = true;
        // change the messages array
        messages = image
            ? [
                  ...messages,
                  {
                      role: "user",
                      type: "image",
                      content: message,
                      image: URL.createObjectURL(image),
                  },
              ]
            : [
                  ...messages,
                  {
                      role: "user",
                      type: "text",
                      content: message,
                  },
              ];
        // if we were scrolled to the bottom
        if (scrolled)
            setTimeout(
                () => (container.scrollTop = container.scrollTopMax),
                40
            );
        // request a response
        let response = image
            ? await fetch(`${url}api/v1/vision`, {
                  method: "POST",
                  body: form,
              }).then((res) => res.json())
            : await fetch(`${url}api/v1/text`, {
                  method: "POST",
                  headers: { "Content-Type": "application/json" },
                  body: JSON.stringify({ input: message }),
              }).then((res) => res.json());
        // turn off sending message
        sendingMessage = false;
        streamingStarted = false;
        // if tts is enabled, fetch the audio wav and make a blob
        let blobUrl = response.tts
            ? await fetch(`${url}ame_speech.wav`)
                  .then((res) => res.blob())
                  .then((blob) => URL.createObjectURL(blob))
            : null;
        // change the last message
        (messages[messages.length - 1].type = response.tts
            ? "augmented"
            : "text"),
            (messages[messages.length - 1].content = response.output),
            (messages[messages.length - 1].audio = response.tts
                ? blobUrl
                : null);
        // reload the messages array
        messages = [...messages];
        // if we were scrolled to the bottom
        if (scrolled)
            setTimeout(
                () => (container.scrollTop = container.scrollTopMax),
                40
            );
    }

    // connect to the server
    async function connect() {
        // set state to connected
        state = "connected";
        // connect to the socket
        socket = new EventSource(`${url}api/v1/sse`);
        // on chunk
        socket.onmessage = (msg) => {
            // parse the data
            let data = JSON.parse(msg.data);
            // if this is the start of a new ai message
            if (messages[messages.length - 1].role === "user") {
                // add the new message
                messages.push({
                    role: "assistant",
                    type: "text",
                    content: data,
                    index: messages.length,
                });
                streamingStarted = true;
                if (scrolled)
                    setTimeout(
                        () => (container.scrollTop = container.scrollTopMax),
                        20
                    );
            }
            // otherwise
            else {
                messages[messages.length - 1].content += data;
                if (scrolled)
                    setTimeout(
                        () => (container.scrollTop = container.scrollTopMax),
                        20
                    );
            }
        };
    }

    // start recording
    async function startRecording() {
        recordingAudio = true;
        mediaRecorder.start();
    }

    // cancel recording
    async function cancelRecording() {
        recordingAudio = false;
        mediaRecorder.canceled = true;
        mediaRecorder.stop();
    }

    // send recording
    async function sendRecording() {
        recordingAudio = false;
        sendingMessage = true;
        mediaRecorder.stop();
    }

    // init shit when the website fully loads
    onMount(async () => {
        state = "disconnected";
        marked.use(markedKatex());
        hljs.addPlugin(new CopyButtonPlugin());
        mediaRecorder = new MediaRecorder(
            await navigator.mediaDevices.getUserMedia({ audio: true }),
            {
                mimeType: "audio/webm",
            }
        );
        // when we stop recording
        mediaRecorder.ondataavailable = async (e) => {
            // push audio chunks
            chunks.push(e.data);
            // if the recording wasn't canceled
            if (!mediaRecorder.canceled) {
                // create a blob from the chunks
                let blob = new Blob(chunks, {
                    type: "audio/webm",
                });
                // add to the messages array
                messages = [
                    ...messages,
                    {
                        role: "user",
                        type: "audio",
                        audio: window.URL.createObjectURL(blob),
                    },
                ];
                // if we were scrolled to the bottom
                if (scrolled)
                    setTimeout(
                        () => (container.scrollTop = container.scrollTopMax),
                        40
                    );
                // make a formdata thing and add the recording
                let formData = new FormData();
                formData.append("recording", blob);
                // get the response from the server
                let response = await fetch(`${url}api/v1/full`, {
                    method: "POST",
                    body: formData,
                }).then((res) => res.json());
                // turn off sending message
                sendingMessage = false;
                streamingStarted = false;
                // if tts is enabled, fetch the audio wav and make a blob
                let blobUrl = response.tts
                    ? await fetch(`${url}ame_speech.wav`)
                          .then((res) => res.blob())
                          .then((blob) => URL.createObjectURL(blob))
                    : null;
                // modify the last message
                messages[messages.length - 2].type = "augmented";
                messages[messages.length - 2].content = response.input;
                // change the last message
                (messages[messages.length - 1].type = response.tts
                    ? "augmented"
                    : "text"),
                    (messages[messages.length - 1].content = response.output),
                    (messages[messages.length - 1].audio = response.tts
                        ? blobUrl
                        : null);
                // reload the messages array
                messages = [...messages];
                // if we were scrolled to the bottom
                if (scrolled)
                    setTimeout(
                        () => (container.scrollTop = container.scrollTopMax),
                        40
                    );
            }
            // reset canceled
            mediaRecorder.canceled = false;
            // clear chunks
            chunks = [];
        };
    });

    $: if (messages.length !== 0 && !streamingStarted)
        setTimeout(hljs.highlightAll, 20);

    function processScroll() {
        scrolled = container.scrollTop === container.scrollTopMax;
    }
</script>

<div
    class="w-screen h-screen bg-ame bg-center bg-cover flex flex-col items-center justify-center text-blue-100"
>
    {#if state === "disconnected"}
        <div
            class="flex flex-grow flex-col items-center justify-center select-none"
        >
            <main
                class="min-w-[25vw] min-h-[30vh] p-6 rounded-xl flex flex-col items-center justify-center {!dev
                    ? 'gap-3'
                    : 'gap-1.5'} border-2 border-blue-100 shadow-lg shadow-blue-300/30"
            >
                <div
                    class="flex flex-col lg:leading-loose md:leading-loose leading-tight text-center items-center {!dev
                        ? 'gap-1.5'
                        : ''}"
                >
                    <h1 class="text-3xl font-bold">
                        梅雨 / Tsuyu{dev ? " (Dev)" : ""}
                    </h1>
                    {#if dev}
                        <p>Please enter the URL of your Tsuyu instance.</p>
                    {:else}
                        <p class="text-wrap w-[75%] leading-tight">
                            Setting a new standard for open-source virtual
                            assistants.
                        </p>
                    {/if}
                </div>
                <div class="flex gap-3">
                    {#if dev}
                        <input
                            bind:value={url}
                            on:keydown={(e) => {
                                if (e.key === "Enter" && !e.shiftKey) {
                                    e.preventDefault();
                                    connect();
                                }
                            }}
                            class="bg-blue-100 text-ame p-4 h-10 align-middle rounded-full focus:outline-none focus:shadow-lg hover:shadow-lg hover:shadow-blue-200/30 focus:shadow-blue-200/30 transition-shadow ease-in-out duration-200"
                        />
                    {/if}
                    <button
                        on:click={connect}
                        class="bg-blue-200 text-ame px-4 h-10 align-middle rounded-full hover:shadow-lg hover:shadow-blue-200/30 transition-shadow ease-in-out duration-200"
                        >Go</button
                    >
                </div>
            </main>
        </div>
    {:else if state === "connected"}
        <div
            class="absolute z-50 flex flex-col top-8 justify-center items-center backdrop-blur-md border-2 align-center border-blue-100 py-2 px-4 rounded-full select-none shadow-lg shadow-blue-300/30"
        >
            <h1 class="text-xl font-bold">梅雨 / Tsuyu{dev ? " (Dev)" : ""}</h1>
        </div>
        <main
            class="flex flex-grow flex-col items-center lg:w-[50vw] md:w-[70vw] w-[90vw] h-full"
        >
            <div
                bind:this={container}
                on:scroll={processScroll}
                class="flex flex-col flex-grow gap-4 overflow-y-scroll w-full"
            >
                <div class="h-1 w-48 shrink-0" />
                {#if messages.length === 0 && !sendingMessage}
                    <h1 class="text-center text-xl pt-20 hyphens-auto">
                        There are no messages yet. Use the textbox to start
                        chatting with Tsuyu!
                    </h1>
                    <p class="hyphens-auto">
                        As a LLaMA-powered chatbot, Tsuyu is highly capable,
                        performing nearly as well as its closed-source
                        competitors.
                    </p>
                    <p class="hyphens-auto">
                        Tsuyu is capable of all of the following:
                    </p>
                    <ul class="mt-[-0.75rem] list-disc list-inside">
                        <li>Processing large amounts of text</li>
                        <li>Generating code</li>
                        <li>Speaking to you</li>
                        <li>Creating tables</li>
                        <li>Formatting messages in Markdown and LaTeX</li>
                        <li>And more!</li>
                    </ul>
                    <p class="hyphens-auto">
                        Please note that the Tsuyu developers are not
                        responsible for any inaccurate or harmful generations.
                        Since users provide their own models, guardrails are
                        typically personalized, and not chosen by the Tsuyu
                        team.
                    </p>
                {:else}
                    {#each messages as message}
                        <div
                            class="message flex flex-col pt-2 pb-4 px-4 rounded-xl border-2 border-blue-100 max-w-[80%] hyphens-auto leading-snug {message.role ===
                            'user'
                                ? 'self-start'
                                : 'self-start bg-rain'}"
                        >
                            {#if message.role === "user"}
                                <div class="align-middle py-2 flex gap-1.5">
                                    <img
                                        class="h-6 w-6 rounded-full bg-blue-100 inline"
                                        src="/you_pfp.png"
                                    />
                                    <span class="font-semibold">You</span>
                                </div>
                            {:else}
                                <div class="align-middle py-2 flex gap-1.5">
                                    <img
                                        class="h-6 w-6 rounded-full bg-blue-100 inline self-center"
                                        src="/ame_pfp.jpg"
                                    />
                                    <span class="font-semibold self-center"
                                        >Tsuyu</span
                                    >
                                    <span class="flex-grow"></span>
                                    <button
                                        class="self-center align-middle hover:text-slate-400 transition-colors ease-in-out duration-200"
                                        on:click={() => {
                                            navigator.clipboard.writeText(
                                                message.content
                                            );
                                            message.copyButton.innerText =
                                                "done";
                                            setTimeout(
                                                () =>
                                                    (message.copyButton.innerText =
                                                        "content_copy"),
                                                2000
                                            );
                                        }}
                                    >
                                        <span
                                            class="material-icons text-base font-bold"
                                            bind:this={message.copyButton}
                                        >
                                            content_copy
                                        </span>
                                    </button>
                                </div>
                            {/if}
                            {#if message.type !== "audio"}
                                <SvelteMarkdown
                                    source={message.content}
                                    options={marked.defaults}
                                    renderers={{
                                        blockKatex: katex,
                                        inlineKatex: katex,
                                    }}
                                />
                            {/if}
                            {#if message.type === "augmented" || message.type === "audio"}
                                <audio
                                    controls
                                    autoplay={messages.length - 1 ===
                                    message.index
                                        ? "autoplay"
                                        : undefined}
                                >
                                    <source
                                        src={message.audio}
                                        type={message.role === "assistant"
                                            ? "audio/wav"
                                            : "audio/webm"}
                                    />
                                    Your browser does not support audio. Please get
                                    one that does.
                                </audio>
                            {/if}
                            {#if message.type === "image"}
                                <a target="_blank" href={message.image}>
                                    <img
                                        class="w-24 h-24 object-cover rounded-xl"
                                        src={message.image}
                                    />
                                </a>
                            {/if}
                        </div>
                    {/each}
                {/if}
                {#if sendingMessage && !streamingStarted}
                    <div class="align-middle">
                        <img
                            class="h-16 w-16 rounded-full inline"
                            src="/spinner.apng"
                        />
                        <span class="text-lg select-none"
                            >Generating response...</span
                        >
                    </div>
                {/if}
                <div class="h-24 w-48 shrink-0" />
            </div>
        </main>
        <div
            class="absolute bottom-8 lg:w-[48vw] md:w-[68vw] w-[88vw] flex flex-col gap-4"
        >
            {#each files as file}
                <div class="w-24 h-24">
                    <img
                        class="object-cover rounded-xl w-full h-full"
                        src={URL.createObjectURL(file)}
                    />
                    <button
                        class="absolute rounded-full left-20 -top-1.5"
                        on:click={() => (files = [])}
                    >
                        <span
                            class="material-icons hover:text-slate-400 transition-colors ease-in-out duration-200"
                            >cancel</span
                        >
                    </button>
                </div>
            {/each}
            <div
                class="{recordingAudio
                    ? 'justify-center p-4 backdrop-blur-md bg-transparent border-2 border-blue-100 rounded-3xl'
                    : ''} flex gap-2"
            >
                {#if !recordingAudio}
                    <textarea
                        autofocus
                        rows="1"
                        style="height: auto"
                        placeholder="Input a message..."
                        class="flex-grow backdrop-blur-md bg-transparent border-2 border-blue-100 p-4 align-middle rounded-3xl focus:outline-none focus:shadow-lg hover:shadow-lg hover:shadow-blue-200/30 focus:shadow-blue-200/30 transition-shadow ease-in-out duration-200"
                        bind:this={textarea}
                        on:input={() => {
                            textarea.style.height = "auto";
                            textarea.style.height = `${textarea.scrollHeight + 4}px`;
                        }}
                        on:keydown={(e) => {
                            if (e.key === "Enter" && !e.shiftKey) {
                                e.preventDefault();
                                sendMessage();
                            }
                        }}
                        disabled={sendingMessage ? "disabled" : undefined}
                    />
                    <button
                        class="hover:text-slate-400 transition-colors ease-in-out duration-200"
                        on:click={startRecording}
                        disabled={sendingMessage ? "disabled" : undefined}
                    >
                        <span class="material-icons">mic</span>
                    </button>
                    <input
                        bind:files
                        bind:this={filePicker}
                        type="file"
                        accept="image/*"
                        class="hidden"
                    />
                    <button
                        class="hover:text-slate-400 transition-colors ease-in-out duration-200"
                        on:click={filePicker.click()}
                        disabled={files.length !== 0 ? "disabled" : undefined}
                    >
                        <span class="material-icons">add_photo_alternate</span>
                    </button>
                {:else}
                    <div
                        class="flex w-full gap-4 place-items-center justify-center"
                    >
                        <div class="flex gap-2 justify-center align-middle">
                            <span
                                class="align-middle material-icons text-red-500 animate-pulse select-none"
                                >mic</span
                            >
                            <span
                                class="align-middle text-red-500 font-semibold"
                                >Recording...</span
                            >
                        </div>
                        <button
                            class="bg-blue-200 text-ame px-4 h-10 align-middle rounded-full hover:shadow-lg hover:shadow-blue-200/30 transition-shadow ease-in-out duration-200"
                            on:click={cancelRecording}
                        >
                            <span class="material-icons align-middle"
                                >cancel</span
                            >
                            <span class="align-middle">Cancel</span>
                        </button>
                        <button
                            class="bg-blue-200 text-ame px-4 h-10 align-middle rounded-full hover:shadow-lg hover:shadow-blue-200/30 transition-shadow ease-in-out duration-200"
                            on:click={sendRecording}
                        >
                            <span class="material-icons align-middle">send</span
                            >
                            <span class="align-middle">Send</span>
                        </button>
                    </div>
                {/if}
            </div>
        </div>
    {/if}
</div>
