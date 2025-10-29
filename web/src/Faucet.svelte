<script>
  import { onMount } from 'svelte';
  import { getAddress } from '@ethersproject/address';
  import { CloudflareProvider } from '@ethersproject/providers';
  import { setDefaults as setToast, toast } from 'bulma-toast';

  let input = null;
  let faucetInfo = {
    account: '0x0000000000000000000000000000000000000000',
    network: 'testnet',
    payout: 1,
    symbol: 'TIA',
    hcaptcha_sitekey: '',
  };

  let mounted = false;
  let hcaptchaLoaded = false;

  onMount(async () => {
    const res = await fetch('/api/info');
    faucetInfo = await res.json();
    mounted = true;
  });

  window.hcaptchaOnLoad = () => {
    hcaptchaLoaded = true;
  };

  $: document.title = `${faucetInfo.symbol} ${capitalize(
    faucetInfo.network,
  )} Faucet`;

  let widgetID;
  $: if (mounted && hcaptchaLoaded) {
    widgetID = window.hcaptcha.render('hcaptcha', {
      sitekey: faucetInfo.hcaptcha_sitekey,
    });
  }

  setToast({
    position: 'bottom-center',
    dismissible: true,
    pauseOnHover: true,
    closeOnClick: false,
    animate: { in: 'fadeIn', out: 'fadeOut' },
  });

  async function handleRequest() {
    let address = input;
    if (address === null) {
      toast({ message: 'input required', type: 'is-warning' });
      return;
    }

    if (address.endsWith('.eth')) {
      try {
        const provider = new CloudflareProvider();
        address = await provider.resolveName(address);
        if (!address) {
          toast({ message: 'invalid ENS name', type: 'is-warning' });
          return;
        }
      } catch (error) {
        toast({ message: error.reason, type: 'is-warning' });
        return;
      }
    }

    try {
      address = getAddress(address);
    } catch (error) {
      toast({ message: error.reason, type: 'is-warning' });
      return;
    }

    try {
      let headers = {
        'Content-Type': 'application/json',
      };

      if (hcaptchaLoaded) {
        const { response } = await window.hcaptcha.execute(widgetID, {
          async: true,
        });
        headers['h-captcha-response'] = response;
      }

      const res = await fetch('/api/claim', {
        method: 'POST',
        headers,
        body: JSON.stringify({
          address,
        }),
      });

      let { msg } = await res.json();
      let type = res.ok ? 'is-success' : 'is-warning';
      toast({ message: msg, type });
    } catch (err) {
      console.error(err);
    }
  }

  function capitalize(str) {
    const lower = str.toLowerCase();
    return str.charAt(0).toUpperCase() + lower.slice(1);
  }
</script>

<svelte:head>
  {#if mounted && faucetInfo.hcaptcha_sitekey}
    <script
      src="https://hcaptcha.com/1/api.js?onload=hcaptchaOnLoad&render=explicit"
      async
      defer
    ></script>
  {/if}
</svelte:head>

<main>
  <section class="hero is-info is-fullheight">
    <div class="wave-container">
      <div class="dots-container">
        {#each Array(20) as _, i}
          <div class="dot" style="--delay: {Math.random() * 5}s; --duration: {15 + Math.random() * 10}s; --x: {Math.random() * 100}%; --y: {Math.random() * 100}%"></div>
        {/each}
      </div>
    </div>
    <div class="hero-head">
      <nav class="navbar">
        <div class="container">
          <div class="navbar-brand">
            <a class="navbar-item" href="../..">
              <span class="icon">
                <i class="fa fa-bath" />
              </span>
              <span><b>{faucetInfo.symbol} Faucet</b></span>
            </a>
          </div>
          <div id="navbarMenu" class="navbar-menu">
            <div class="navbar-end">
            </div>
          </div>
        </div>
      </nav>
    </div>

    <div class="hero-body">
      <div class="container has-text-centered">
        <div class="column is-8 is-offset-2">
          <h1 class="title">
            Receive {faucetInfo.payout}
            {faucetInfo.symbol} per 24 hours
          </h1>
          <div id="hcaptcha" data-size="invisible"></div>
          <div class="box">
            <div class="field is-grouped">
              <p class="control is-expanded">
                <input
                  bind:value={input}
                  class="input is-rounded"
                  type="text"
                  placeholder="Enter your address"
                />
              </p>
              <p class="control">
                <button
                  on:click={handleRequest}
                  class="button is-primary is-rounded"
                >
                  Request
                </button>
              </p>
            </div>
          </div>
        </div>
      </div>
    </div>
  </section>
</main>

<style>
  :global(html) {
    font-size: 16px;
  }

  :global(body) {
    font-size: 16px;
  }

  .hero.is-info {
    background: url('/flora.png') center center / cover no-repeat;
    position: relative;
    overflow: hidden;
  }

  .wave-container {
    position: absolute;
    width: 100%;
    height: 100%;
    top: 0;
    left: 0;
    overflow: hidden;
  }

  .dots-container {
    position: absolute;
    width: 100%;
    height: 100%;
    top: 0;
    left: 0;
  }

  .dot {
    position: absolute;
    width: 4px;
    height: 4px;
    background: rgba(255, 255, 255, 0.6);
    border-radius: 50%;
    left: var(--x);
    top: var(--y);
    animation: float var(--duration) ease-in-out var(--delay) infinite,
               pulse 2s ease-in-out var(--delay) infinite;
  }

  @keyframes float {
    0%, 100% {
      transform: translateY(0) translateX(0);
    }
    25% {
      transform: translateY(-20px) translateX(10px);
    }
    50% {
      transform: translateY(10px) translateX(-10px);
    }
    75% {
      transform: translateY(-10px) translateX(5px);
    }
  }

  @keyframes pulse {
    0%, 100% {
      opacity: 0.6;
      transform: scale(1);
    }
    50% {
      opacity: 1;
      transform: scale(1.5);
    }
  }

  .hero-head, .hero-body {
    position: relative;
    z-index: 1;
  }

  .hero .title {
    font-size: 72px;
    font-weight: bold;
    text-shadow: 3px 3px 8px rgba(0, 0, 0, 0.9), 0 0 20px rgba(0, 0, 0, 0.5);
    color: white;
    margin-bottom: 60px;
  }

  .hero .subtitle {
    padding: 2rem 0;
    line-height: 1.5;
    font-size: 32px;
    text-shadow: 3px 3px 8px rgba(0, 0, 0, 0.9), 0 0 20px rgba(0, 0, 0, 0.5);
    color: white;
    font-weight: 500;
  }

  .box {
    border-radius: 20px;
    padding: 30px;
    background: rgba(255, 255, 255, 0.98);
    box-shadow: 0 4px 20px rgba(0, 0, 0, 0.25);
  }

  .field.is-grouped {
    display: flex;
    align-items: center;
    gap: 20px;
  }

  .control.is-expanded {
    flex: 1;
  }

  .input {
    font-size: 32px !important;
    height: 80px;
    padding: 20px 30px;
    border: 3px solid #ddd;
    background: white;
    font-weight: 500;
    line-height: 1.2;
  }

  .input:focus {
    border-color: #00d1b2;
    outline: none;
    box-shadow: 0 0 0 4px rgba(0, 209, 178, 0.15);
  }

  .input::placeholder {
    font-size: 18px;
    color: #999;
  }

  .button {
    font-size: 24px;
    height: 80px;
    padding: 0 48px;
    font-weight: 700;
    white-space: nowrap;
  }

  /* Responsive adjustments */
  @media (max-width: 768px) {
    .hero .title {
      font-size: 40px !important;
      margin-bottom: 30px;
    }
    .hero .subtitle {
      font-size: 24px !important;
    }
    .input {
      font-size: 24px !important;
      height: 60px;
    }
    .button {
      font-size: 20px;
      height: 60px;
      padding: 0 30px;
    }
  }
</style>
