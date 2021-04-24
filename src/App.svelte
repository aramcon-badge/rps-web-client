<script lang="ts">
  /// <reference types="web-bluetooth" />

  const UART_SERVICE_UUID = '6e400001-b5a3-f393-e0a9-e50e24dcca9e';
  const UART_CHARACTERISTIC_RX = '6e400002-b5a3-f393-e0a9-e50e24dcca9e';
  const UART_CHARACTERISTIC_TX = '6e400003-b5a3-f393-e0a9-e50e24dcca9e';

  enum GameState {
    Idle,
    Connecting,
    GameStart,
    SentChoice,
    Error,
  }

  let gameState = GameState.Idle;
  let errorMessage = '';
  let opponentChoice = [];
  let remoteRx: BluetoothRemoteGATTCharacteristic;

  const rpsOptions = ['Rock', 'Paper', 'Scissors'];

  function addChecksum(bytes: Uint8Array) {
    const result = new Uint8Array(bytes.length + 1);
    result.set(bytes);
    result[bytes.length] = ~bytes.reduce((a, b) => a + b, 0);
    return result;
  }

  function buttonPacket(index: number, pressed: boolean) {
    const packet = `!B${index.toString()}${pressed ? '1' : '0'}`;
    return addChecksum(new TextEncoder().encode(packet));
  }

  async function sendButton(choice: number) {
    try {
      const packet = buttonPacket(choice, true);
      await remoteRx.writeValue(packet);
    } catch (err) {
      console.error(err);
      gameState = GameState.Error;
      errorMessage = err.toString();
    }
  }

  async function sendGameChoice(choice: number) {
    gameState = GameState.SentChoice;
    await sendButton(choice);
  }

  async function sendRematchChoice(rematch: boolean) {
    if (rematch) {
      gameState = GameState.GameStart;
      await sendButton(2);
    } else {
      gameState = GameState.Idle;
      await sendButton(1);
      remoteRx.service.device.gatt.disconnect();
    }
  }

  async function connect() {
    gameState = GameState.Connecting;
    try {
      const device = await navigator.bluetooth.requestDevice({
        filters: [{ services: [UART_SERVICE_UUID] }],
      });
      await device.gatt.connect();
      const service = await device.gatt.getPrimaryService(UART_SERVICE_UUID);
      const remoteTx = await service.getCharacteristic(UART_CHARACTERISTIC_TX);
      remoteTx.addEventListener('characteristicvaluechanged', () => {
        const data = new TextDecoder().decode(remoteTx.value);
        console.log('got', data);
        if (/^!B[123]1/.test(data)) {
          opponentChoice.push(data[2]);
        }
      });
      await remoteTx.startNotifications();
      remoteRx = await service.getCharacteristic(UART_CHARACTERISTIC_RX);
      gameState = GameState.GameStart;
    } catch (err) {
      console.error(err);
      gameState = GameState.Error;
      errorMessage = err.toString();
    }
  }
</script>

<main>
  <h1>RPS Client</h1>
  {#if gameState === GameState.Idle || gameState === GameState.Error}
    <button on:click={connect}>Connect</button>
  {/if}
  {#if gameState === GameState.Connecting}
    Connecting...
  {/if}
  {#if gameState === GameState.GameStart}
    Your selection?
    <button on:click={() => sendGameChoice(1)}>Rock</button>
    <button on:click={() => sendGameChoice(2)}>Paper</button>
    <button on:click={() => sendGameChoice(3)}>Scissors</button>
  {/if}
  {#if gameState === GameState.SentChoice}
    <div>
      opponentChoice: {rpsOptions[opponentChoice[0] - 1]}
    </div>
    Play again?
    <button on:click={() => sendRematchChoice(false)}>No</button>
    <button on:click={() => sendRematchChoice(true)}>Yes</button>
  {/if}
  {#if gameState === GameState.Error}
    <div class="error">
      Error: {errorMessage}
    </div>
  {/if}
</main>

<style>
  main {
    text-align: center;
    padding: 1em;
    max-width: 480px;
    margin: 0 auto;
  }

  h1 {
    font-weight: 100;
  }

  .error {
    color: #ff3e00;
  }
</style>
