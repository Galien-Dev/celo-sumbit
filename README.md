#  Building an NFT Auction Smart Contract in Solidity on celo blockchain

## Table of contents

- [Introduction](#introduction)
- [Prerequisites](#prerequisites)
- [Step 1: Setting up the Truffle Project](#step-1-setting-up-the-truffle-project)
- [Step 2: Setting up the Smart Contract](#step-2-setting-up-the-smart-contract)
- [Step 3: Implementing the Auction Functionality](#step-3-implementing-the-auction-sunctionality)
- [Step 4: Testing the Smart Contract](#step-4-testing-the-smart-contract)
- [Step 5: Running the Tests](#step-5-running-the-tests)
- [Conclusion](#conclusion)

## Introduction
Non-fungible tokens (NFTs) have gained significant popularity as they provide a unique way to own, transfer, 
and trade digital assets. NFTs enable the creation and sale of one-of-a-kind digital items like artwork,
collectibles, and virtual real estate. To facilitate this new form of commerce, smart contracts play a crucial role. 
In this tutorial, we will explore the implementation of an NFT auction smart contract in Solidity, specifically for the Celo blockchain using the Alfajores testnet.

## Prerequisites

Before we begin, ensure that you have the following:

- Basic knowledge of Solidity programming language.
- An understanding of ERC-721 standard for NFTs.
- Familiarity with the OpenZeppelin library.
- Truffle installed and configured.

## Step 1: Setting up the Truffle Project
1. Open a terminal and navigate to the directory where you want to create your Truffle project run the commands below.
```bash
mkdir nft-auction
cd nft-auction
```
2. Run the following command to initialize a new Truffle projec
```bash
truffle init
```
This will create a basic Truffle project structure with the necessary files.
3. Open the truffle-config.js file and configure the Celo network settings:
```javascript
module.exports = {
  networks: {
    alfajores: {
      provider: () => new HDWalletProvider(mnemonic, `https://alfajores-forno.celo-testnet.org`),
      network_id: 44787,
      gas: 8000000,
    },
  },
  compilers: {
    solc: {
      version: "0.8.9",
      settings: {
        optimizer: {
          enabled: true,
          runs: 200,
        },
      },
    },
  },
};

```
Replace `mnemonic` with your Celo testnet mnemonic.

## Step 2: Setting up the Smart Contract
First, let's set up the basic structure of our NFT auction smart contract.
Open your preferred code editor and create a new Solidity file called NFTAuction.sol.
Copy and paste the following code into the file:

```solidity
pragma solidity ^0.8.0;

import "@openzeppelin/contracts/token/ERC721/ERC721.sol";

contract NFTAuction is ERC721 {
    // Contract variables and mappings
    
    constructor() ERC721("NFTAuction", "NFTA") {
        // Contract initialization
    }
    
    // Contract functions
}
```
In this contract, we import the `ERC721` contract from the OpenZeppelin library and inherit from it.
The `constructor` function is called during contract deployment and can be used for initialization.

## Step 3: Implementing the Auction Functionality

First, let's set up the basic structure of our NFT auction smart contract.
Open your preferred code editor and create a new Solidity file called NFTAuction.sol.
Copy and paste the following code into the file:

```solidity

// SPDX-License-Identifier: UNLICENSED
pragma solidity ^0.8.9;

import '@openzeppelin/contracts/token/ERC721/extensions/ERC721URIStorage.sol';
import '@openzeppelin/contracts/security/ReentrancyGuard.sol';


contract NFTAuction is ERC721URIStorage, ReentrancyGuard {

    uint256 public tokenCounter;
    uint256 public listingCounter;

    uint8 public constant STATUS_OPEN = 1;
    uint8 public constant STATUS_DONE = 2;

    uint256 public minAuctionIncrement = 10; // 10 percent

    struct Listing {
        address seller;
        uint256 tokenId;
        uint256 price; // display price
        uint256 netPrice; // actual price
        uint256 startAt;
        uint256 endAt; 
        uint8 status;
    }

    event Minted(address indexed minter, uint256 nftID, string uri);
    event AuctionCreated(uint256 listingId, address indexed seller, uint256 price, uint256 tokenId, uint256 startAt, uint256 endAt);
    event BidCreated(uint256 listingId, address indexed bidder, uint256 bid);
    event AuctionCompleted(uint256 listingId, address indexed seller, address indexed bidder, uint256 bid);
    event WithdrawBid(uint256 listingId, address indexed bidder, uint256 bid);

    mapping(uint256 => Listing) public listings;
    mapping(uint256 => mapping(address => uint256)) public bids;
    mapping(uint256 => address) public highestBidder;

    constructor() ERC721("AmazonNFT", "ANFT") {
        tokenCounter = 0;
        listingCounter = 0;
    }


    function mint(string memory tokenURI, address minterAddress) public returns (uint256 tokenIdentity) {
        require(bytes(tokenURI).length > 0, "Token URI must not be empty");
        require(minterAddress != address(0), "Minter address cannot be zero");
        tokenCounter++;
        uint256 tokenId = tokenCounter;

        _safeMint(minterAddress, tokenId);
        _setTokenURI(tokenId, tokenURI);

        emit Minted(minterAddress, tokenId, tokenURI);

        return tokenId;
    }

    function createAuctionListing (uint256 price, uint256 tokenId, uint256 durationInSeconds) public returns (uint256 listingID) {
        require(durationInSeconds > 0, "Duration must be greater than zero");
    require(price > 0, "Price must be greater than zero");
    require(tokenId > 0, "Token ID must be greater than zero");
        listingCounter++;
        uint256 listingId = listingCounter;

        uint256 startAt = block.timestamp;
        uint256 endAt = startAt + durationInSeconds;

        listings[listingId] = Listing({
            seller: msg.sender,
            tokenId: tokenId,
            price: price,
            netPrice: price,
            status: STATUS_OPEN,
            startAt: startAt,
            endAt: endAt
        });

        _transfer(msg.sender, address(this), tokenId);

        emit AuctionCreated(listingId, msg.sender, price, tokenId, startAt, endAt);

        return listingId;
    }

    function bid(uint256 listingId) public payable nonReentrant {
        require(isAuctionOpen(listingId), "Auction has ended");
        Listing storage listing = listings[listingId];
        require(msg.sender != listing.seller, "Cannot bid on what you own");
        require(msg.value > 0, "Bid value must be greater than zero");

        uint256 newBid = bids[listingId][msg.sender] + msg.value;
        require(newBid >= listing.price, "Cannot bid below the latest bidding price");

        bids[listingId][msg.sender] += msg.value;
        highestBidder[listingId] = msg.sender;

        uint256 incentive = listing.price * minAuctionIncrement / 100;
        listing.price = listing.price + incentive;

        emit BidCreated(listingId, msg.sender, newBid);
    }



    function completeAuction(uint256 listingId) public payable nonReentrant {
        require(!isAuctionOpen(listingId), "Auction is still open");

        Listing storage listing = listings[listingId];
        address winner = highestBidder[listingId]; 
        require(
            msg.sender == listing.seller || msg.sender == winner, 
            "Only the seller or the winner can complete the auction"
        );

        if (winner != address(0)) {
            _transfer(address(this), winner, listing.tokenId);

            uint256 amount = bids[listingId][winner]; 
            require(amount > 0, "Invalid bid amount");
            bids[listingId][winner] = 0;
            _transferFund(payable(listing.seller), amount);
        } else {
            require(listing.seller != address(0), "Invalid seller address");
            _transfer(address(this), listing.seller, listing.tokenId);
        }

        listing.status = STATUS_DONE;

        emit AuctionCompleted(listingId, listing.seller, winner, bids[listingId][winner]);
    }


    function withdrawBid(uint256 listingId) public payable nonReentrant {
        require(isAuctionExpired(listingId), "Auction must be ended");
        require(highestBidder[listingId] != msg.sender, "Highest bidder cannot withdraw bid");

        uint256 balance = bids[listingId][msg.sender];
        require(balance > 0, "No available bid to withdraw");
        bids[listingId][msg.sender] = 0;
        _transferFund(payable(msg.sender), balance);

        emit WithdrawBid(listingId, msg.sender, balance);
    }


    function isAuctionOpen(uint256 id) public view returns (bool) {
        require(id > 0 && id <= listingCounter, "Invalid listing ID");

        return
            listings[id].status == STATUS_OPEN &&
            listings[id].endAt > block.timestamp;
    }



    function isAuctionExpired(uint256 id) public view returns (bool) {
        require(id > 0 && id <= listingCounter, "Invalid listing ID");   
        return listings[id].endAt <= block.timestamp;
    }



    function _transferFund(address payable to, uint256 amount) internal {
        require(amount > 0, "Invalid transfer amount");
        require(to != address(0), "Invalid recipient address");
    
        (bool transferSent, ) = to.call{value: amount}("");
        require(transferSent, "Failed to send Ether");
    }


}

```
The above contract has a `tokenCounter` and `listingCounter` to keep track of the number of NFTs and auctions, respectively.
The contract also has several constant values such as the minimum auction increment and the auction status, which can be open or done. 
The structure Listing is used to store information about each auction, such as the seller, tokenId, price, start time, and end time.

The contract also has several mappings to keep track of the auction listings, bids, and highest bidder.
The `mint` function is used to create a new NFT and `createAuctionListing` is used to create a new auction.
The  `bid` function allows a user to place or update a bid on an auction and the  completeAuction  function allows
the auction to be completed by either the seller or the highest bidder. The  `withdrawBid` function allows
a user to withdraw their bid if the auction has ended and they are not the highest bidder.

The `isAuctionOpen` checks if the auction is still running based on the status and timestamp of the auction and 
The `isAuctionExpired` method does check if the auction is closed. The `_transferFund method` is used to transfer funds (Ether) from one address to another.

### Step 4: Testing the Smart Contract
Let us create the testing deployment script file and paste the following codes:
```javascritp
const { constants } = require("@openzeppelin/test-helpers");
import { ethers } from "hardhat";
import { expect } from "chai";

const nftAuctionDeployer = async () => {
  let nftAuctionContract;

  nftAuctionContract = await ethers.getContractFactory("NFTAuction");
  nftAuctionContract = await nftAuctionContract.deploy();

  expect(nftAuctionContract.address).to.not.equal(constants.ZERO_ADDRESS);

  return {
    nftAuctionContract,
  };
};

export default nftAuctionDeployer;
```

Add another file to contain helper functions and paste this code snippet:

```javascritp
import { ethers } from "hardhat";

export const weiToEth = (balance: any) => {
  return ethers.utils.formatEther(balance);
};

export const ethToWei = (balance: any) => {
  return ethers.utils.parseEther(balance);
};

function toFixed(num: any, fixed: number) {
  const re = new RegExp("^-?\\d+(?:.\\d{0," + (fixed || -1) + "})?");
  return Number(num.toString().match(re)[0]);
}

export function toUnits(balance: any) {
  return toFixed(ethers.utils.formatEther(balance), 4);
}

export const increaseTime = async (seconds: number) => {
  await ethers.provider.send("evm_increaseTime", [seconds]);
  await ethers.provider.send("evm_mine", []);
};
```

Finally, create a file to contain our test cases and paste the following code snippet:

```javascript
import { ethers } from "hardhat";
import { expect } from "chai";
import { NFTAuction } from "../typechain-types/contracts";
import nftAuctionDeployer from "./helpers/nft-auction.deployer";
import { SignerWithAddress } from "@nomiclabs/hardhat-ethers/signers";
import { ethToWei, increaseTime, toUnits } from "./helpers/test.utils";

describe.only("NFT Auction Test Cases", () => {
  let nftAuctionContract: NFTAuction,
    userMinter: SignerWithAddress,
    bidder: SignerWithAddress,
    bidder2: SignerWithAddress,
    tokenId: number,
    listingId: number;

  before(async () => {
    [userMinter, bidder, bidder2] = await ethers.getSigners();

    ({ nftAuctionContract } = await nftAuctionDeployer());
  });

  it("It should mint an NFT", async () => {
    // The user mints the NFT
    const mintTrx: any = await nftAuctionContract
      .connect(userMinter)
      .mint("ape-vv-ss", userMinter.address);

    const trxReceipt = await mintTrx.wait();
    tokenId = Number(trxReceipt?.events[0]?.args["tokenId"]);

    // NFT ID must be 1
    expect(tokenId).to.be.equal(1);

    // User minter must be the owner of NFT 1
    expect(userMinter.address).to.be.equal(
      await nftAuctionContract.ownerOf(tokenId)
    );

    // User minter must own NFT 1
    expect(
      Number(await nftAuctionContract.balanceOf(userMinter.address))
    ).to.be.equal(1);
  });

  it("It should auction listing", async () => {
    // The user create auction listing to run for 2 days
    const auctionListingTrx: any = await nftAuctionContract
      .connect(userMinter)
      .createAuctionListing(ethToWei("0.3"), tokenId, 86400 * 2);

    const trxReceipt = await auctionListingTrx.wait();
    listingId = Number(trxReceipt?.events[2]?.args["listingId"]);

    // Listing ID must be 1
    expect(listingId).to.be.equal(1);

    // The current contract must become escrow account
    expect(nftAuctionContract.address).to.be.equal(
      await nftAuctionContract.ownerOf(tokenId)
    );
  });

  it("It should place bids, update bid, complete auction & withdraw funds", async () => {
    const bidderBalanceBeforeBid = await ethers.provider.getBalance(
      bidder.address
    );
    const bidder2BalanceBeforeBid = await ethers.provider.getBalance(
      bidder2.address
    );

    // bidder 1 & 2 place their bids
    await nftAuctionContract
      .connect(bidder)
      .bid(listingId, { value: ethToWei("0.6") });

    await nftAuctionContract
      .connect(bidder2)
      .bid(listingId, { value: ethToWei("0.7") });

    // bidder 1 updates his bid in order to become the highest bidder
    await nftAuctionContract
      .connect(bidder)
      .bid(listingId, { value: ethToWei("0.2") });

    const bidderBalanceAfterBid = await ethers.provider.getBalance(
      bidder.address
    );
    const bidder2BalanceAfterBid = await ethers.provider.getBalance(
      bidder2.address
    );

    expect(toUnits(bidderBalanceBeforeBid)).to.be.greaterThan(
      toUnits(bidderBalanceAfterBid)
    );
    expect(toUnits(bidder2BalanceBeforeBid)).to.be.greaterThan(
      toUnits(bidder2BalanceAfterBid)
    );

    // After 3 days the auction has ended
    await increaseTime(3 * 86400);

    const nftOwnerBalanceBeforeAuctionCompletion =
      await ethers.provider.getBalance(userMinter.address);

    // The highest bidder complete the auction
    await nftAuctionContract.connect(bidder).completeAuction(listingId);

    const nftOwnerBalanceAfterAuctionCompletion =
      await ethers.provider.getBalance(userMinter.address);

    // The NFT owner collects funds from the highest bidder
    expect(toUnits(nftOwnerBalanceAfterAuctionCompletion)).to.be.greaterThan(
      toUnits(nftOwnerBalanceBeforeAuctionCompletion)
    );

    // The Highest bidder becomes the new NFT owner
    expect(await nftAuctionContract.ownerOf(tokenId)).to.be.equal(
      bidder.address
    );

    expect(
      Number(await nftAuctionContract.balanceOf(userMinter.address))
    ).to.be.equal(0);
    expect(
      Number(await nftAuctionContract.balanceOf(bidder.address))
    ).to.be.equal(1);

    // The bidder 2 collects his funds after the auction has ended
    const withdrawTrx = await nftAuctionContract
      .connect(bidder2)
      .withdrawBid(listingId);

    const withdrawReceipt: any = await withdrawTrx.wait();
    const withdrawBidder = withdrawReceipt.events[0]?.args["bidder"];
    const withdrawBid = withdrawReceipt.events[0]?.args["bid"];

    expect(withdrawBidder).to.be.equal(bidder2.address);
    expect(toUnits(withdrawBid)).to.be.equal(0.7);
  });
});
```

The above test file contains the following test cases:

- **Minting an NFT**: a user mints an NFT using the mint function and verifies that the correct
NFT ID is generated and the user is set as the owner of the NFT.
- **Auction listing**`: the user creates an auction listing for the NFT, and the test verifies that the correct
listing ID is generated and the NFT is transferred to the contract as the escrow account.
- **Placing bids, updating bids, completing the auction, and withdrawing funds**: two bidders place bids for the NFT, and then
-  one of the bidders updates their bid to become the highest bidder. The test checks that the bidders’ balances decrease after 
-  they place their bids. After 3 days, the highest bidder completes the auction, and the NFT owner collects the funds. The test verifies 
-  that the NFT owner’s balance increases after the auction is completed.

## Step 5: Running the Tests
To run the tests, follow these steps:

1. Install Truffle framework if you haven't already: npm install -g truffle
2. Open a terminal and navigate to the directory containing the NFTAuctionTest.sol file.
3. Run the tests using the Truffle command: truffle test
Ensure that your tests pass successfully without any errors or failures.

## Conclusion
In this tutorial, we explored the implementation of an NFT auction smart contract in Solidity.
We covered the setup of the smart contract, the implementation of the auction functionality,
and the creation of test cases. Building on this foundation, you can further enhance the smart contract
to suit your specific requirements and deploy it on the Celo blockchain using the Alfajores testnet.
