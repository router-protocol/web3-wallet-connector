<template>
  <div>
    <button @click="connectWalletWagmi">ConnectWagmi</button>
    <button @click="switchToRouterWagmi">SwitchWagmi</button>
    <button @click="mint">Mint</button>
    <button @click="getNftDataRouter">GetNftDataRouter</button>
    <button @click="getNftDataEVM">GetNftDataEVM</button>
    <button @click="transferFromRouterToEVM">TransferFromRouterToEVM</button>
    <button @click="transferFromEVMToRouter">TransferFromEVMToRouter</button>
  </div>
</template>

<script>
import axios from "axios";
import Web3 from "web3";

import {
  getEndpointsForNetwork,
  Network,
  TxRestClient,
  toUtf8,
  ChainGrpcWasmApi,
  executeQueryInjected,
  getRouterSignerAddress,
} from "@routerprotocol/router-chain-sdk-ts";

import characterData from "../pages/data.json";
const abi = require("./abi.json");

import {
  EthereumClient,
  w3mConnectors,
  w3mProvider,
} from "@web3modal/ethereum";
import { Web3Modal } from "@web3modal/html";
import { configureChains, createConfig, watchAccount } from "@wagmi/core";
import { arbitrum, mainnet, polygon } from "@wagmi/core/chains";
const projectId = "be1c34299e26b37abbc852b6ed59e3a2";
import { getWalletClient } from "@wagmi/core";

export const router = {
  id: 9601,
  name: "Router Testnet",
  network: "router",
  nativeCurrency: {
    decimals: 18,
    name: "Router",
    symbol: "Route",
  },
  rpcUrls: {
    public: { http: ["https://evm.rpc.testnet-eu.routerchain.dev/"] },
    default: { http: ["https://evm.rpc.testnet-eu.routerchain.dev/"] },
  },
  blockExplorers: {
    etherscan: {
      name: "scan",
      url: "https://alpha-explorer-ui.routerprotocol.com",
    },
    default: {
      name: "scan",
      url: "https://alpha-explorer-ui.routerprotocol.com",
    },
  },
};

const chains = [arbitrum, mainnet, polygon, router];

const { publicClient } = configureChains(chains, [w3mProvider({ projectId })]);
const wagmiConfig = createConfig({
  autoConnect: true,
  connectors: w3mConnectors({ projectId, chains }),
  publicClient,
});
const ethereumClient = new EthereumClient(wagmiConfig, chains);
const web3modal = new Web3Modal({ projectId }, ethereumClient);

var accountUser = null;
const unwatch = watchAccount((account) => {
  console.log(account);
  accountUser = account;
});

export default {
  name: "IndexPage",
  data() {
    return {
      routerContractAddress:
        "router10lges24q56ssqv2zmz9ayelz8anrpkywfmpjtutpjd58853knanswxfxe9",
      mumbaiContractAddress: "0xe403Dd7245f3Cb1E48235e00f224a0D18601F57B",
      routerChainId: "0x2581",
      web3: null,
      characterInfo: characterData.characterInfo,
      contract: null,
      contractABI: abi,
      userAddress: "",
      injectedSigner: "",
    };
  },
  created() {},
  methods: {
    async switchToRouterWagmi() {
      try {
        await ethereumClient.switchNetwork({ chainId: 9601 });
      } catch (err) {
        console.log(err);
      }
    },

    async connectWalletWagmi() {
      try {
        await web3modal.openModal();
      } catch (err) {
        console.log(err);
      }
    },

    async setUserAddressAndClient() {
      try {
        console.log(ethereumClient.getNetwork().chain.id)
        await this.switchToRouterWagmi()
        this.userAddress = ethereumClient.getAccount().address;
        const walletClient = await getWalletClient();
        this.injectedSigner = walletClient;
        if(ethereumClient.getNetwork().chain.id != 9601) {
          alert('You have to switch a network in your wallet manually. Go to your wallet and switch to Router network and then try again.')
          return
        }
      } catch (err) {
        console.log(err);
      }
    },

    // is already minted func
    async isAlreadyMinted() {
      try {
        const endpoint = getEndpointsForNetwork(Network.Testnet);
        const wasmClient = new ChainGrpcWasmApi(endpoint.grpcEndpoint);

        const request = {
          address: this.routerContractAddress,
          queryData: toUtf8(
            JSON.stringify({
              extension: {
                msg: {
                  is_already_minted: {
                    owner: getRouterSignerAddress(this.userAddress),
                  },
                },
              },
            })
          ),
        };

        const fetchSmartContractStateResult =
          await wasmClient.fetchSmartContractState(
            request.address,
            request.queryData
          );
        console.log(fetchSmartContractStateResult.data);
        return fetchSmartContractStateResult.data;
      } catch (err) {
        console.log("isAlreadyMinted() exception: ", err);
        alert(
          "Error! Please, contact us and provide this message so we can solve it: isAlreadyMinted() exception: ",
          err
        );
      }
      return null;
    },

    // mint function
    async mint() {
      if (!accountUser.isConnected) {
        try {
          await this.connectWalletWagmi();
        } catch (err) {
          console.log(err);
        }
        return;
      }

      try {
        await this.setUserAddressAndClient();

        if (await this.isAlreadyMinted()) {
          alert("You have already minted! You cannot mint more than 1 token.");
          return;
        }
      } catch (err) {
        console.log("mint() checking isAlreadyMinted() exception: ", err);
        alert(
          "Error! Please, contact us and provide this message so we can solve it: mint() checking isAlreadyMinted() exception: ",
          err
        );
        return;
      }

      var tokenUri = "none";
      var signature = "none";

      try {
        const jsonBody = this.characterInfo;
        var response = await axios.post(
          "http://3.106.130.141/store_state",
          jsonBody
        );
        tokenUri = response.data.link;
        signature = response.data.signature;
        console.log("response from backend", response);
      } catch (err) {
        console.log("mint() exception error in loading to backend: ", err);
        alert("Error in loading data to backend. Please, contact us.");
        return;
      }

      try {
        const restClient = new TxRestClient(
          getEndpointsForNetwork(Network.Testnet).lcdEndpoint
        );

        const jsonMsg = {
          extension: {
            msg: {
              mint_token: {
                token_uri: tokenUri,
                signature: signature,
                // owner: getRouterSignerAddress(window.ethereum.selectedAddress),
              },
            },
          },
        };

        const tx = await executeQueryInjected({
          networkEnv: Network.Testnet,
          contractAddress: this.routerContractAddress,
          executeMsg: jsonMsg,
          nodeUrl: getEndpointsForNetwork(Network.Testnet).lcdEndpoint,
          ethereumAddress: this.userAddress,
          injectedSigner: this.injectedSigner,
        });

        const txHash = tx.tx_response.txhash;
        const txResponse = await restClient.waitTxBroadcast(txHash);
        console.log("mint() txResponse: ", txResponse);
        alert("the token is minted ", txResponse);
        return true;
      } catch (err) {
        console.log("mint() exception: ", err);
        if (
          err ==
          "Error: Transaction was not included in a block before timeout of 30000ms"
        ) {
          alert(
            "The transaction will be included in the block later, please wait 1-2 mins and update the page and you will see your NFT"
          );
        } else {
          alert(
            "An error occured in mint(), most possibly because you don't have enough funds to pay for gas, if you do, please contact us, details: " +
              err
          );
        }
      }
      return false;
    },

    // get token uri of an nft on router chain
    async getNftDataRouter() {
      try {
        const tokenId = await this.getNftsRouter();
        const endpoint = getEndpointsForNetwork(Network.Testnet);
        const wasmClient = new ChainGrpcWasmApi(endpoint.grpcEndpoint);

        const request = {
          address: this.routerContractAddress,
          queryData: toUtf8(
            JSON.stringify({ nft_info: { token_id: tokenId } })
          ),
        };

        const fetchSmartContractStateResult =
          await wasmClient.fetchSmartContractState(
            request.address,
            request.queryData
          );

        console.log(
          "get NFT info on Router: fetchSmartContractStateResult =>",
          fetchSmartContractStateResult.data.token_uri
        );
        alert("NFT Data: " + fetchSmartContractStateResult.data.token_uri);
      } catch (err) {
        console.log("getNftDataRouter exception: ", err);
        alert("getNftDataRouter exception, details: " + err);
      }
    },

    // func to get nft id on router chain
    async getNftsRouter() {
      if (!accountUser.isConnected) {
        try {
          await this.connectWalletWagmi();
        } catch (err) {
          console.log(err);
        }
        return;
      }
      await this.setUserAddressAndClient();
      try {
        const endpoint = getEndpointsForNetwork(Network.Testnet);
        const wasmClient = new ChainGrpcWasmApi(endpoint.grpcEndpoint);

        const request = {
          address: this.routerContractAddress,
          queryData: toUtf8(
            JSON.stringify({
              tokens: {
                owner: getRouterSignerAddress(this.userAddress),
                start_after: "0",
                limit: 1,
              },
            })
          ),
        };

        const fetchSmartContractStateResult =
          await wasmClient.fetchSmartContractState(
            request.address,
            request.queryData
          );
        console.log(
          "get Nfts on Router: fetchSmartContractStateResult =>",
          fetchSmartContractStateResult
        );
        return fetchSmartContractStateResult.data.tokens[0];
      } catch (err) {
        console.log("getNftsRouter exception: " + err);
      }
    },

    // connect wallet to EVM
    async connectWalletMumbai() {
      if (window.ethereum) {
        try {
          // Request account access
          const accounts = await window.ethereum.request({
            method: "eth_requestAccounts",
          });

          // Check if the current network is Router Chain
          const chainId = await window.ethereum.request({
            method: "eth_chainId",
          });
          if (chainId !== "0x13881") {
            try {
              // Try to switch to Router Chain
              await window.ethereum.request({
                method: "wallet_switchEthereumChain",
                params: [{ chainId: "0x13881" }],
              });
            } catch (switchError) {
              // This error code indicates that the chain has not been added to MetaMask.
              if (switchError.code === 4902) {
                try {
                  await window.ethereum.request({
                    method: "wallet_addEthereumChain",
                    params: [
                      {
                        chainId: "0x13881", // Mumbai testnet chain ID
                        chainName: "Polygon Mumbai Testnet",
                        nativeCurrency: {
                          name: "MATIC",
                          symbol: "MATIC", // 2-6 characters long
                          decimals: 18,
                        },
                        rpcUrls: [
                          "https://endpoints.omniatech.io/v1/matic/mumbai/public",
                        ], // RPC URL for Mumbai testnet
                        blockExplorerUrls: ["https://mumbai.polygonscan.com"], // Block explorer URL for Mumbai testnet
                      },
                    ],
                  });
                } catch (addError) {
                  // handle "add" error
                }
              }
              // handle other "switch" errors
            }
          }

          // We don't know window.web3 version, so we use our own instance of Web3
          // with the injected provider given by MetaMask
          this.web3 = new Web3(window.ethereum);

          console.log(`Connected with the account: ${accounts[0]}`);
        } catch (error) {
          // User denied account access...
          console.error("User denied account access");
        }
      }
      // Legacy dapp browsers...
      else if (window.web3) {
        this.web3 = new Web3(window.web3.currentProvider);
        // Acccounts always exposed
        console.log(
          `Connected with the account: ${window.web3.currentProvider.selectedAddress}`
        );
      }
      // Non-dapp browsers...
      else {
        console.log(
          "Non-Ethereum browser detected. You should consider trying MetaMask!"
        );
      }
    },

    // connect evm contract
    async connectContract() {
      this.contract = new this.web3.eth.Contract(
        this.contractABI,
        this.mumbaiContractAddress
      );
    },

    // get token id from evm
    async getTokenIdEVM() {
      try {
        const tokenId = await this.contract.methods
          .tokenOfOwnerByIndex(this.userAddress, 0)
          .call();
        console.log("getTokenId() on EVM: ", tokenId);
        return tokenId;
      } catch (error) {
        console.error("Error in getTokenId:", error);
      }
    },

    // get nft token uri from evm
    async getTokenUriEVM(tokenId) {
      try {
        const tokenURI = await this.contract.methods.tokenURI(tokenId).call();
        console.log("getTokenURI() on EVM: ", tokenURI);
        return tokenURI;
      } catch (error) {
        console.error("Error in getTokenURI:", error);
      }
    },

    // get nft data from evm
    async getNftDataEVM() {
      try {
        await this.connectWalletMumbai();
        await this.connectContract();
        const tokenId = await this.getTokenIdEVM();
        const tokenURI = await this.getTokenUriEVM(tokenId);
        console.log(
          "getNftsEVM successful, tokenId: " +
            tokenId +
            " tokenURI: " +
            tokenURI
        );
        alert(
          "getNftsEVM successful, tokenId: " +
            tokenId +
            " tokenURI: " +
            tokenURI
        );
      } catch (err) {
        console.log("getNftDataEVM exception: ", err);
        alert("getNftDataEVM exception, details: " + err);
      }
    },

    // crosschain transfer from Router to EVM
    async transferFromRouterToEVM() {
      if (!accountUser.isConnected) {
        try {
          await this.connectWalletWagmi();
        } catch (err) {
          console.log(err);
        }
        return;
      }
      await this.setUserAddressAndClient();

      const tokenId = await this.getNftsRouter();
      if (tokenId == null || tokenId == undefined) {
        alert("you dont have an nft on router");
        return;
      }

      const restClient = new TxRestClient(
        getEndpointsForNetwork(Network.Testnet).lcdEndpoint
      );

      const jsonMsg = {
        extension: {
          msg: {
            transfer_cross_chain: {
              dst_chain_id: "80001",
              token_id: parseInt(tokenId),
              recipient: this.userAddress,
              request_metadata: {
                dest_gas_limit: 300000,
                dest_gas_price: 30000000000,
                ack_gas_limit: 300000,
                ack_gas_price: 30000000000,
                relayer_fee: "10000000000",
                ack_type: "NoAck",
                is_read_call: false,
                asm_address: "",
              },
            },
          },
        },
      };
      console.log("jsonMSG: ", jsonMsg);

      try {
        const tx = await executeQueryInjected({
          networkEnv: Network.Testnet,
          contractAddress: this.routerContractAddress,
          executeMsg: jsonMsg,
          nodeUrl: getEndpointsForNetwork(Network.Testnet).lcdEndpoint,
          ethereumAddress: this.userAddress,
          injectedSigner: this.injectedSigner,
        });

        const txHash = tx.tx_response.txhash;
        const txResponse = await restClient.waitTxBroadcast(txHash);
        console.log(txResponse);
        alert("Transfer details: " + txResponse);
      } catch (err) {
        console.log(err);
        alert(
          "An error occured, most possibly because you don't have enough funds to pay for gas, details: " +
            err
        );
      }
    },

    async transferFromEVMToRouter() {
      try {
        await this.connectWalletMumbai();
        await this.connectContract();
        // defining the parameters for transferCrossChain function
        let chainName = "RouterTestnet"; // replace with your chain name
        let tokenId = await this.getTokenIdEVM();
        // invoking the transferCrossChain function
        await this.contract.methods
          .transferCrossChain(chainName, parseInt(tokenId))
          .send({
            from: this.userAddress,
            value: this.web3.utils.toWei("0", "ether"),
          })
          .on("transactionHash", function (hash) {
            console.log("transactionHash", hash);
            alert("transaction sent! hash:" + hash);
          });
      } catch (err) {
        console.log(err);
        alert("error in transfer from EVM to Router: " + err);
      }
    },
  },
};
</script>
