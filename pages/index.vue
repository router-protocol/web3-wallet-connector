<template>
  <div>
    <button @click="connectWalletWagmi">ConnectWagmi</button>
    <button @click="switchNetworkInMetamask">switch</button>
    <button @click="connectWalletRouter">Connect</button>
    <button @click="connectWalletRouterWC2">Wallet Connect</button>
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
  getEthereumChainIdForNetwork,
  getNetworkType,
} from "@routerprotocol/router-chain-sdk-ts";

import characterData from "../pages/data.json";
const abi = require("./abi.json");

import { createConfig, configureChains, mainnet } from "@wagmi/core";
import { publicProvider } from "@wagmi/core/providers/public";

import { connect, fetchEnsName } from "@wagmi/core";
import { InjectedConnector } from "@wagmi/core/connectors/injected";
import { WalletConnectConnector } from "wagmi/connectors/walletConnect";

const configRouter = {
  chainId:
    "0x" + getEthereumChainIdForNetwork(getNetworkType("testnet")).toString(16),
  chainName: `Router Testnet`,
  nativeCurrency: {
    name: "Route",
    symbol: "ROUTE",
    decimals: 18,
  },
  rpcUrls: [getEndpointsForNetwork(getNetworkType("testnet")).rpcEndpoint],
  blockExplorerUrls: ["https://explorer.testnet.routerchain.dev"],
};

const { chains, publicClient, webSocketPublicClient } = configureChains(
  [mainnet],
  [publicProvider()]
);

const config = createConfig({
  autoConnect: true,
  publicClient,
  webSocketPublicClient,
});

const GetInjectedProviders = () => {
  if (typeof window !== "undefined") {
    if (window.ethereum) {
      return window.ethereum;
    }
    if (window.frontier) {
      return window.frontier.ethereum;
    }
    if (window.cosmostation) {
      return window.cosmostation.ethereum;
    }
    return undefined;
  }
  return undefined;
};

const injectedConnector = new InjectedConnector({
  options: {
    name: "My Injected Wallet",
    getProvider: GetInjectedProviders,
  },
});

const walletConnectConnector = new WalletConnectConnector({
  options: {
    projectId: "be1c34299e26b37abbc852b6ed59e3a2",
    qrModalOptions: {},
  },
});

const checkIfRouterChainId = (chainId) => {
  switch (chainId) {
    case "9600":
      return true;
    case "9601":
      return true;
    case "9602":
      return true;
    case "9603":
      return true;
    case "9604":
      return true;
    case "9605":
      return true;
    default:
      return false;
  }
};


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
    };
  },
  created() {},
  methods: {
    async switchNetworkInMetamask() {
      if (window.ethereum) {
        // Check if the current network is Router Chain
        const chainId = await window.ethereum.request({
          method: "eth_chainId",
        });
        if (chainId !== this.routerChainId) {
          try {
            // Try to switch to Router Chain
            await window.ethereum.request({
              method: "wallet_switchEthereumChain",
              params: [{ chainId: this.routerChainId }],
            });
          } catch (switchError) {
            // This error code indicates that the chain has not been added to MetaMask.
            
              try {
                await window.ethereum.request({
                  method: "wallet_addEthereumChain",
                  params: [
                    {
                      chainId: this.routerChainId,
                      chainName: "Router testnet",
                      nativeCurrency: {
                        name: "Route",
                        symbol: "Route", // 2-6 characters long
                        decimals: 18,
                      },
                      rpcUrls: ["https://evm.rpc.testnet.routerchain.dev"],
                      blockExplorerUrls: [
                        "https://alpha-explorer-ui.routerprotocol.com",
                      ],
                    },
                  ],
                });
              } catch (addError) {
                console.log("connectWallet() exception: ", addError);
                alert(
                  "An error in switching networks, please contact us, details: ",
                  addError
                );
                // handle "add" error
              }
           
          }
        }
      } 
    },
    async connectWalletWagmi() {
      try {
        var _walletClient = null
        if(window.ethereum) {
          await injectedConnector.connect();
          _walletClient = await injectedConnector.getWalletClient();
        } else {
          await walletConnectConnector.connect();
          _walletClient = await walletConnectConnector.getWalletClient();
        }
        
        const _address = (await _walletClient.getAddresses())[0];

        alert(_address)
        this.switchNetworkInMetamask();
      } catch (err) {
        console.log(err);
      }
    },

    async connectWalletRouterWC() {
      let provider;

      if (window.ethereum) {
        console.log("desktop");
        provider = window.ethereum;
        try {
          // Request account access
          const accounts = await provider.request({
            method: "eth_requestAccounts",
          });
          // ... rest of your code
        } catch (error) {
          // User denied account access...
          console.error("User denied account access");
        }
      }
      // Legacy dapp browsers...
      else if (window.web3) {
        provider = window.web3.currentProvider;
        // Acccounts always exposed
        console.log(
          `Connected with the account: ${window.web3.currentProvider.selectedAddress}`
        );
      }
      // If no injected provider is detected, fall back to WalletConnect
      else {
        console.log("mobile");
        provider = new WalletConnectProvider({
          infuraId: "ba540ca359744e14a31171bc87df6ea4", // Required
          // bridge: "https://bridge.walletconnect.org", // Optional. If not set, it defaults to the value above
          // qrcodeModalOptions: {
          //   mobileLinks: [
          //     "rainbow",
          //     "metamask",
          //     // Add more wallet links
          //   ],
          // },
        });

        try {
          // Enable session (triggers QR Code modal)
          await provider.enable();
        } catch (e) {
          console.error("Could not get a wallet connection", e);
          return;
        }

        console.log(`Connected with the account: ${provider.selectedAddress}`);
      }

      // We don't know window.web3 version, so we use our own instance of Web3
      // with the injected provider given by MetaMask or WalletConnect
      this.web3 = new Web3(provider);
    },
    // connect wallet to router function
    async connectWalletRouter() {
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
          if (chainId !== this.routerChainId) {
            try {
              // Try to switch to Router Chain
              await window.ethereum.request({
                method: "wallet_switchEthereumChain",
                params: [{ chainId: this.routerChainId }],
              });
            } catch (switchError) {
              // This error code indicates that the chain has not been added to MetaMask.
              if (switchError.code === 4902) {
                try {
                  await window.ethereum.request({
                    method: "wallet_addEthereumChain",
                    params: [
                      {
                        chainId: this.routerChainId,
                        chainName: "Router testnet",
                        nativeCurrency: {
                          name: "Route",
                          symbol: "Route", // 2-6 characters long
                          decimals: 18,
                        },
                        rpcUrls: ["https://evm.rpc.testnet.routerchain.dev"],
                        blockExplorerUrls: [
                          "https://alpha-explorer-ui.routerprotocol.com",
                        ],
                      },
                    ],
                  });
                } catch (addError) {
                  console.log("connectWallet() exception: ", addError);
                  alert(
                    "An error in switching networks, please contact us, details: ",
                    addError
                  );
                  // handle "add" error
                }
              } else {
                console.log(
                  "connectWallet() switchError exception: ",
                  switchError
                );
                alert(
                  "An error in switching network, please contact us, details: ",
                  switchError
                );
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
          //alert('Error! You denied account access')
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
        alert("No wallet detected. Please, install MetaMask");
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
                    owner: getRouterSignerAddress(
                      window.ethereum.selectedAddress
                    ),
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
      try {
        await this.connectWalletRouter();
      } catch (err) {
        console.log("mint() connectWallet() exception: ", err);
        alert(
          "An error occured, please contact us; mint() while connectWallet() exception: ",
          err
        );
        return;
      }

      try {
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
          ethereumAddress: window.ethereum.selectedAddress,
          injectedSigner: window.ethereum,
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
      try {
        const endpoint = getEndpointsForNetwork(Network.Testnet);
        const wasmClient = new ChainGrpcWasmApi(endpoint.grpcEndpoint);

        const request = {
          address: this.routerContractAddress,
          queryData: toUtf8(
            JSON.stringify({
              tokens: {
                owner: getRouterSignerAddress(window.ethereum.selectedAddress),
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
          .tokenOfOwnerByIndex(window.ethereum.selectedAddress, 0)
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
      try {
        await this.connectWalletRouter();
      } catch (err) {
        console.log("mint() connectWallet() exception: ", err);
        alert(
          "An error occured, please contact us; mint() while connectWallet() exception: ",
          err
        );
        return;
      }
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
              recipient: window.ethereum.selectedAddress,
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
          ethereumAddress: window.ethereum.selectedAddress,
          injectedSigner: window.ethereum,
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
            from: window.ethereum.selectedAddress,
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
