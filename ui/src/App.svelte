<script>
  import { onMount, onDestroy, afterUpdate } from 'svelte';
  import { writable, get } from 'svelte/store';
  // @ts-ignore
  import hljs from 'highlight.js';
  // @ts-ignore
  import markdownit from 'markdown-it';

  let md = new markdownit();
  let messages = writable([]);
  let systemMessage = '';
  let userInput = '';
  let autoScrollState = true;
  let modelName = 'gpt-3.5-turbo';
  let modelToggle = false;

  function renderMarkdown(content) {
    return md.render(content);
  }

  function highlightCode(element) {
    const codeElements = element.querySelectorAll("pre code");
    codeElements.forEach((codeElement) => {
      hljs.highlightElement(codeElement);
    });
  }

  function updateSystemMessage(systemMessage) {
    let systemMessageIndex = get(messages).findIndex((message) => message.role === "system");
    if (systemMessageIndex !== -1) {
      get(messages).splice(systemMessageIndex, 1);
    }
    if (systemMessage) {
      messages.update(msgs => [...msgs, { role: "system", content: systemMessage }]);
    }
  }

  async function postRequest() {
    return await fetch("http://localhost:8000/gpt4", {
      method: "POST",
      body: JSON.stringify({
        messages: get(messages),
        model_type: modelName,
      }),
      headers: {
        "Content-Type": "application/json",
      },
    });
  }

  async function handleResponse(response) {
    const reader = response.body.getReader();
    const decoder = new TextDecoder("utf-8");
    let assistantMessage = "";

    while (true) {
      const { value, done } = await reader.read();
      if (done) {
        messages.update(msgs => [...msgs, { role: "assistant", content: assistantMessage }]);
        break;
      }
      const text = decoder.decode(value);
      assistantMessage += text;
    }
  }

  async function handleSubmit(event) {
    event.preventDefault();

    updateSystemMessage(systemMessage);
    messages.update(msgs => [...msgs, { role: "user", content: userInput }]);
    userInput = '';
    const response = await postRequest();
    handleResponse(response);
  }

  function handleModelToggle() {
    modelToggle = !modelToggle;
    modelName = modelToggle ? 'gpt-4' : 'gpt-3.5-turbo';
  }

  afterUpdate(() => {
    const messageDiv = document.querySelectorAll('.message');
    messageDiv.forEach(div => highlightCode(div));
  });

  let chatMessagesDiv;
  onMount(() => {
    chatMessagesDiv = document.getElementById("chat-messages");
    chatMessagesDiv.addEventListener("scroll", function () {
      const isAtBottom =
        chatMessagesDiv.scrollHeight - chatMessagesDiv.clientHeight <=
        chatMessagesDiv.scrollTop + 1;

      autoScrollState = isAtBottom;
    });
  });

  onDestroy(() => {
    chatMessagesDiv.removeEventListener("scroll");
  });
</script>

<div class="menubar">
  <input type="text" class="input" bind:value={systemMessage} placeholder="Enter system message">
  <label class="slider-container">
    <input type="checkbox" bind:checked={modelToggle} on:change={handleModelToggle}>
    <span class="slider-track">
      <span class="model-label">{modelToggle ? 'GPT-4' : 'GPT-3.5'}</span>
      <div class="slider"></div>
    </span>
  </label> 
</div>
<div class="chat-container">
  <h1 class="heading">Canvas GPT</h1>
  <div class="chat-wrapper">
    <div bind:this={chatMessagesDiv} id="chat-messages" class="chat-box">
      {#each $messages as message (message.content)}
        <div class={message.role === "user" ? "message user-message" : "message assistant-message"}>
          <p>{@html renderMarkdown(message.content)}</p>
        </div>
      {/each}
    </div>
    <form id="chat-form" class="input-form" on:submit|preventDefault={handleSubmit}>
      <textarea bind:value={userInput} class="input" placeholder="Type your message..."></textarea>
      <button class="button" type="submit">Send</button>
    </form>
  </div>
</div>
