<script>
  import { Socket } from "phoenix";
  import MessageBox from "./MessageBox.svelte";

  let messages = [];
  let textInput = "";
  const MAX_MESSAGES = 10;
  const ENTER_KEYCODE = 13;
  const WS_ENDPOINT = "ws://localhost:4000/socket";

  let socket = new Socket(WS_ENDPOINT, { params: {} });

  socket.connect();

  let channel = socket.channel("room:lobby", {});
  channel
    .join()
    .receive("ok", (resp) => {
      console.log("Joined successfully", resp);
    })
    .receive("error", (resp) => {
      console.log("Unable to join", resp);
    });

  channel.on("new_msg", (payload) => {
    const message = payload.body;
    const sliced = messages.slice(
      Math.max(messages.length - MAX_MESSAGES + 1, 0)
    );
    messages = [...sliced, message];
  });

  function handleKeyPress(e) {
    if (e.keyCode == ENTER_KEYCODE) {
      handleSubmit(e);
    }
  }

  function handleSubmit(e) {
    // Send message via ws. Disable input on start, enable on response. Clear text on success.
    // For now, simulate messages population directly.
    if (textInput !== "") {
      channel.push("new_msg", { body: textInput });

      textInput = "";
    }
  }
</script>

<MessageBox {messages} />

<input type="text" bind:value={textInput} on:keypress={handleKeyPress} />
<button on:click={handleSubmit}>Submit</button>
