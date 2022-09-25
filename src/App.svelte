<script>

  import {BigNumber, ethers, utils} from "ethers";
  import {pwcCoinAbi, pwcShopAbi, pwcCoinAddress, pwcShopAddress} from './constants/constants.js'
  import {onDestroy, onMount} from "svelte";
  import Input from "./Input.svelte";

  let currentAccount;
  let transferValue = "";
  let targetAddress = "";
  let pwcTokenBuyAmount;
  let pwcTokenSellAmount;
  let pwcTokenBalance = 0;
  $: readableBalance = utils.formatEther(pwcTokenBalance.toString());
  let tokenPrice = 0;
  $: readablePrice = utils.formatEther(tokenPrice.toString());

  let addToShopAmount = 0
  let addToShopPrice = 0;

  let tickets = [];

  onMount(async () => {

    if (typeof window.ethereum === "undefined") {
      setTimeout(() => {
        alert("You need to have Metamask installed.");
      }, 1000)
    } else {

      window.ethereum.on("chainChanged", handleChainChanged)
      window.ethereum.on("accountsChanged", handleAccountsChanged)
      window.ethereum.on("disconnect", handleDisconnect)

      await listenToEvents();
    }
  })

  onDestroy(() => {
    window.ethereum.removeListener("chainChanged", handleChainChanged)
    window.ethereum.removeListener("accountsChanged", handleAccountsChanged)
    window.ethereum.removeListener("disconnect", handleDisconnect)
  })

  const handleAccountsChanged = async () => {
    const provider = new ethers.providers.Web3Provider(window.ethereum);
    const addresses = await provider.listAccounts();
    if (addresses.length === 0) {
      console.log("Please connect to Metamask");
      currentAccount = null;
    } else if (addresses[0] !== currentAccount) {
      currentAccount = addresses[0];
      await getPwCTokenBalance();
      await getPwCTokenPrice();
    }
  };

  const handleChainChanged = () => {
    window.location.reload();
  }

  const handleDisconnect = () => {
    currentAccount = null;
    window.location.reload();
  }

  const connectWallet = async () => {
    try {
      const accounts = await window.ethereum.request({method: "eth_requestAccounts"});
      currentAccount = accounts[0];
      await getPwCTokenBalance();
      await getPwCTokenPrice();
      await getTickets();
    } catch (err) {
      console.log(err);
    }
  }

  const disconnectWallet = () => {
    currentAccount = null;
  };

  const getContract = (contractAddress, abi) => {
    const provider = new ethers.providers.Web3Provider(window.ethereum);
    const signer = provider.getSigner();
    return new ethers.Contract(contractAddress, abi, signer);
  }

  const getPwCTokenBalance = async () => {
    if (typeof window.ethereum !== "undefined") {
      const contract = getContract(pwcCoinAddress, pwcCoinAbi);
      try {
        pwcTokenBalance = await contract.balanceOf(currentAccount);
      } catch (err) {
        console.log(err)
      }
    }
  }

  const getPwCTokenPrice = async () => {
    if (typeof window.ethereum !== "undefined") {
      const contract = getContract(pwcCoinAddress, pwcCoinAbi);
      try {
        tokenPrice = await contract.TOKEN_PRICE();
      } catch (err) {
        console.log(err)
      }
    }
  }

  const transferPWCToken = async () => {
    if (typeof window.ethereum !== "undefined") {
      const contract = getContract(pwcCoinAddress, pwcCoinAbi)

      try {
        const valueToTransfer = BigNumber.from(transferValue).mul(BigNumber.from(10).pow(BigNumber.from(18)));
        const transaction = await contract.transfer(targetAddress, valueToTransfer)
        await transaction.wait();
        await getPwCTokenBalance();
        await getPwCTokenPrice();
      } catch (err) {
        console.log(err)
      }
    }
  }

  const increaseAllowance = async (allowanceValue) => {
    if (typeof window.ethereum !== "undefined") {
      const contract = getContract(pwcCoinAddress, pwcCoinAbi)
      try {
        const increaseAllowanceBy = BigNumber.from(allowanceValue).mul(BigNumber.from(10).pow(BigNumber.from(18)));
        const transaction = await contract.increaseAllowance(pwcShopAddress, increaseAllowanceBy)
        await transaction.wait();
      } catch (err) {
        console.log(err)
      }
    }

  }

  const buyTicket = async (ticket) => {
    if (typeof window.ethereum !== "undefined") {
      const contract = getContract(pwcShopAddress, pwcShopAbi);
      await increaseAllowance(ticket.price);

      try {
        const transaction = await contract.buyTicket(ticket.id);
        await transaction.wait();
        await getTickets();
      } catch (err) {
        console.log(err)
      }
    }
  }

  async function listenToEvents() {
    console.log('listening to events')
    if (typeof window.ethereum !== 'undefined') {
      const [account] = await window.ethereum.request({method: 'eth_requestAccounts'})
      const provider = new ethers.providers.Web3Provider(window.ethereum);
      const contract = new ethers.Contract(pwcCoinAddress, pwcCoinAbi, provider);

      contract.on("Transfer", (from, to, value) => {
        console.log("from: " + from);
        console.log("to: " + to);
        console.log("value: " + value);
      })

      // contract.queryFilter("Transfer", 0, "latest").then((a) => {
      // 	console.log("a")
      // 	console.log(a);
      // })
    }
  }

  const buyPwcToken = async () => {
    if (typeof window.ethereum !== "undefined") {
      const contract = getContract(pwcCoinAddress, pwcCoinAbi);
      try {
        tokenPrice = await contract.TOKEN_PRICE();
        const etherToSend = (pwcTokenBuyAmount * readablePrice).toString();
        const options = {
          value: ethers.utils.parseEther(etherToSend)
        }
        const buyTx = await contract.buyTokens(pwcTokenBuyAmount, options);
        await buyTx.wait();

        await getPwCTokenBalance();
      } catch (err) {
        console.log(err)
      }
    }
  }

  const sellPwcToken = async () => {
    if (typeof window.ethereum !== "undefined") {
      const contract = getContract(pwcCoinAddress, pwcCoinAbi);
      try {
        const sellTx = await contract.sellTokens(pwcTokenSellAmount);
        await sellTx.wait();

        await getPwCTokenBalance();
      } catch (err) {
        console.log(err)
      }
    }
  }

  const addTicketsToShop = async () => {
    if (typeof window.ethereum !== "undefined") {
      const contract = getContract(pwcShopAddress, pwcShopAbi);
      try {
        const priceWithDecimals = BigNumber.from(addToShopPrice).mul(BigNumber.from(10).pow(18)).toString()
        const transaction = await contract.addToShop(addToShopAmount, priceWithDecimals);
        await transaction.wait();

        await getTickets();
      } catch (err) {
        console.log(err)
      }
    }
  }

  const getTickets = async () => {
    if (typeof window.ethereum !== "undefined") {
      const contract = getContract(pwcShopAddress, pwcShopAbi);
      try {
        tickets = await contract.getTickets();
      } catch (err) {
        console.log(err)
      }
    }
  }

  const setForSale = async (id, price) => {
    if (typeof window.ethereum !== "undefined") {
      const contract = getContract(pwcShopAddress, pwcShopAbi);
      try {
        const priceWithDecimals = BigNumber.from(price).mul(BigNumber.from(10).pow(18)).toString()
        const transaction = await contract.setForSale(id.toString(), priceWithDecimals.toString());
        await transaction.wait();
        await getTickets();
      } catch (err) {
        console.log(err)
      }
    }
  }

</script>

<main>
  <h1>PwC Token</h1>
  <h2>Current wallet address:</h2>
  <h2>{currentAccount}</h2>
  <div>
    <button class="btn btn-light" on:click={connectWallet}>Connect Wallet</button>
    <button class="btn btn-danger" on:click={disconnectWallet}>Disconnect Wallet</button>
  </div>
  <hr>
  <div>
    <h3>PwC Token balance: {readableBalance}</h3>
    <h4>PwC Token price: {readablePrice}</h4>
  </div>
  <div>
    <button class="btn btn-light" on:click={buyPwcToken}>Buy PwC Token</button>
    <input type="number" bind:value={pwcTokenBuyAmount}>
  </div>
  <div>
    <button class="btn btn-light" on:click={sellPwcToken}>Sell PwC Token</button>
    <input type="number" bind:value={pwcTokenSellAmount}>
  </div>

  <div style="margin-top: 5px">
    <button class="btn btn-light" on:click={transferPWCToken}>Send PwC Token</button>
    <input type="text" bind:value={targetAddress} placeholder="address">
    <input type="number" bind:value={transferValue} placeholder="amount">
  </div>
  <hr>
  <div>
    <button class="btn btn-light" on:click={addTicketsToShop}>Add tickets to shop</button>
    <input type="number" bind:value={addToShopAmount} placeholder="amount">
    <input type="number" bind:value={addToShopPrice} placeholder="price">
  </div>

  <div>
    <h2>Tickets</h2>
    <div class="container">
      <div class="row">
        <div class="col-12">
          <table class="table">
            <thead>
            <tr>
              <th scope="col">#</th>
              <th scope="col">owner</th>
              <th scope="col">price</th>
              <th scope="col">is for sale</th>
              <th scope="col">is valid</th>
            </tr>
            </thead>
            <tbody>
            {#each tickets as ticket}
              <tr>
                <th scope="row">{ticket.id}</th>
                <td>{ticket.owner}</td>
                <td>{utils.formatEther(ticket.price.toString())}</td>
                <td>{ticket.forSale}</td>
                <td>{ticket.valid}</td>
                {#if ticket.forSale && ticket.owner.toLowerCase() !== currentAccount?.toLowerCase()}
                  <td>
                    <button class="btn btn-light" on:click={buyTicket(ticket)}>Buy</button>
                  </td>
                {/if}
                {#if ticket.owner.toLowerCase() === currentAccount?.toLowerCase()  }
                  <td>
                    <Input ticket="{ticket}" action="{setForSale}"/>
                  </td>
                {/if}
              </tr>
            {/each}
            </tbody>
          </table>
        </div>
      </div>
    </div>
  </div>
</main>

<style>
  main {
    text-align: center;
    padding: 1em;
    max-width: 240px;
    margin: 0 auto;
  }

  h1 {
    color: #ff3e00;
    text-transform: uppercase;
    font-size: 4em;
    font-weight: 100;
  }

  @media (min-width: 640px) {
    main {
      max-width: none;
    }
  }
  button {
    border: solid black 1px
  }

</style>
