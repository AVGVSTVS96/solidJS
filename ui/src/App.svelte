<script>
  import { onMount, onDestroy } from 'svelte';
  import { writable } from 'svelte/store';
  import { get } from 'svelte/store';
  import hljs from 'highlight.js';
  import markdownit from 'markdown-it';

  // Variables
  let chatMessagesDiv;
  let userInputElem;
  let systemMessageElem;
  let modelToggle = false;
  let modelName = 'gpt-3.5-turbo';
  let systemMessage = '';
  let userInput = '';
  const messages = writable([]);
  let autoScrollState = true;
  
  const md = new markdownit();

  // Helper functions
  const renderMarkdown = (content) => md.render(content);

  function highlightCode(element) {
    const codeElements = element.querySelectorAll("pre code");
    codeElements.forEach((codeElement) => {
      hljs.highlightElement(codeElement);
    });
  }

  function autoScroll() {
    if (autoScrollState) {
      chatMessagesDiv.scrollTop = chatMessagesDiv.scrollHeight;
    }
  }

  async function handleResponse(response, messageText) {
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
      messageText.innerHTML = renderMarkdown(assistantMessage).trim();
      highlightCode(messageText);
      autoScroll();
    }
  }

  async function postRequest() {
    const messagesList = get(messages);
    return await fetch("/gpt4", {
      method: "POST",
      body: JSON.stringify({
        messages: messagesList,
        model_type: modelName,
      }),
      headers: {
        "Content-Type": "application/json",
      },
    });
  }

  // Event handlers
  async function handleFormSubmit(event) {
    event.preventDefault();

    userInput = userInputElem.value.trim();
    systemMessage = systemMessageElem.value.trim();

    messages.update(msgs => [...msgs, { role: "user", content: userInput }]);
    userInputElem.value = "";

    const response = await postRequest();

    handleResponse(response, userInput);
  }

  function handleModelToggle() {
  if (modelToggle) {
    modelName = "gpt-4";
  } else {
    modelName = "gpt-3.5-turbo";
  }
}


  onMount(() => {
    chatMessagesDiv.addEventListener("scroll", function () {
      const isAtBottom =
        chatMessagesDiv.scrollHeight - chatMessagesDiv.clientHeight <=
        chatMessagesDiv.scrollTop + 1;
  
      autoScrollState = isAtBottom;
    });
  });

  onDestroy(() => {
    chatMessagesDiv.removeEventListener('scroll');
  });

</script>

<svelte:head>
  <title>GPT-4 API Chat</title>
  <link rel="stylesheet" href="/static/styles.css">
  <link rel="stylesheet" href="https://cdnjs.cloudflare.com/ajax/libs/highlight.js/11.7.0/styles/tokyo-night-dark.min.css">
</svelte:head>

<div class="menubar">
  <input type="text" class="input" bind:this={systemMessageElem} placeholder="Enter system message">
  <label class="slider-container">
    <input type="checkbox" bind:this={modelToggleElem} bind:checked={modelToggle} on:change={handleModelToggle} autocomplete="off">
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
      {#each $messages as message}
        <div class={message.role === "user" ? "message user-message" : "message assistant-message"}>
          <p>{@html renderMarkdown(message.content)}</p>
        </div>
      {/each}
    </div>
    <form id="chat-form" class="input-form" on:submit|preventDefault={handleFormSubmit}>
      <textarea bind:this={userInputElem} class="input" placeholder="Type your message..."></textarea>
      <button class="button" type="submit">Send</button>
    </form>
  </div>
</div>


<style>

</style>
