# Sealed Bid Auction Factory
This is an update to the [sealed-bid auction](https://github.com/baedrik/SCRT-sealed-bid-auction) secret contract running on Secret Network.  The new implementation incorporates the use of a factory contract to allow someone to view only the active auctions or only the closed auctions.  They may also view only the auctions that they have created, the auctions in which they have active bids, and the auctions they have won if they have created a viewing key with the factory contract.  If they have created a viewing key, the factory will automatically set the viewing key with every active auction they have bids with to allow the bidder to view their bid information directly from the auction.

Although the original auction contract will no longer be updated, the repo will remain available so that people may use it as an example for writing secret contracts.  The original repo includes an in depth [WALKTHROUGH](https://github.com/baedrik/SCRT-sealed-bid-auction/blob/master/WALKTHROUGH.md) as well as an explanation of [CALLING_OTHER_CONTRACTS](https://github.com/baedrik/SCRT-sealed-bid-auction/blob/master/CALLING_OTHER_CONTRACTS.md).

## Creating a New Auction
First you must give the factory an allowance to consign the tokens for sale to the new auction's escrow:
```sh
secretcli tx compute execute *sale_tokens_contract_address* '{"increase_allowance":{"spender":"*factory_contract_address*","amount":"*amount_being_sold_in_smallest_denomination_of_sale_token*"}}' --from *your_key_alias_or_addr* --gas 150000 -y
```
Then you can create the auction with:
```sh
secretcli tx compute execute --label *factory_contract_label* '{"create_auction":{"label":"*your_auction_name*","sell_contract":{"code_hash":"*sale_tokens_code_hash*","address":"*sale_tokens_contract_address*"},"bid_contract":{"code_hash":"*bid_tokens_code_hash*","address":"*bid_tokens_contract_address*"},"sell_amount":"*amount_being_sold_in_smallest_denomination_of_sale_token*","minimum_bid":"*minimum_accepted_bid_in_smallest_denomination_of_bid_token*","description":"*optional_text_description*"}}' --from *your_key_alias_or_addr* --gas 600000 -y
```
You can find a contract's code hash with
```sh
secretcli q compute contract-hash *contract_address*
```
Copy it without the 0x prefix and surround it with quotes in the instantiate command.

When the factory creates the new auction it will send the tokens you are putting up for sale to the auction's escrow.  If you did not give the factory sufficient allowance, or if your token balance is less than the sale amount, the auction will not be created.

The description field is optional.  It will accept a free-form text string (best to avoid using double-quotes).  One possible use would be to list the approximate date that you plan to finalize the auction.  In a sealed bid auction, a pre-defined end date is not necessary.  It is necessary in an open ascending bid auction because bidders need to know when the auction will close so that they can monitor if they are winning and bid higher if they are not.  Because in a sealed bid auction, no one knows if they are the highest bidder until after the auction ends, the bidder has no further actions after placing his bid.  For this reason, the auction owner can finalize the auction at any time.  If at any point a bidder no longer wants to wait for the owner to finalize the auction, he can retract his bid and have his bid tokens returned.  For this reason, it might benefit the auction owner to give an approximate end date in the description so that his highest bid doesn't get retracted before he decides to close the auction.  If user consensus would like to have an end date implemented, in which no bids will be accepted after such time, the owner can not finalize the auction before the end date, and afterwards, anyone can close the auction, it can be included.

The auction will not allow a sale amount of 0

The auction will not currently allow the sale contract address to be the same as the bid contract address, because there is no reason to swap different amounts of the same fungible token.  When the SNIP-721 spec is more fleshed out, this will probably be changed to allow for the exchanging of different NFT token IDs regardless of whether they are part of the same NFT contract or not.

## Create a Viewing Key
You can have the factory generate a new viewing key with:
``` sh
secretcli tx compute execute --label *factory_contract_label* '{"create_viewing_key":{"entropy":"*Some arbitrary string used as entropy in generating the random viewing key*"}}' --from *your_key_alias_or_addr* --gas 200000 -y
```
or you can set the viewing key with
``` sh
secretcli tx compute execute --label *factory_contract_label* '{"set_viewing_key":{"key":"*The viewing key you want*","padding":"Optional string used to pad the message length so it does not leak the length of the key"}}' --from *your_key_alias_or_addr* --gas 200000 -y
```
A viewing key allows you to query the factory contract for a list of only the auctions you have interacted with.  It also allows you to view your bid information by querying the individual auctions.  It is recommended to use `create_viewing_key` to set your viewing key instead of using `set_viewing_key` because `create_viewing_key` will generate a complex key, whereas `set_viewing_key` will just accept whatever key it is given.  `set_viewing_key` is provided if a UI would like to generate the viewing key itself.  If you are developing a UI and are using `set_viewing_key`, also use the `padding` field so that the length of the message does not leak information about the length of the viewing key.

## Placing Bids
To place a bid, the bidder should Send the tokens to the individual auction contract address with
```sh
secretcli tx compute execute *bid_tokens_contract_address* '{"send": {"recipient": "*auction_contract_address*", "amount": "*bid_amount_in_smallest_denomination_of_bidding_token*"}}' --from *your_key_alias_or_addr* --gas 500000 -y
```
The tokens bid will be placed in escrow until the auction has concluded or you call retract\_bid to retract your bid and have all tokens returned.  You may retract your bid at any time before the auction ends. You may only have one active bid at a time.  If you place more than one bid, the smallest bid will be returned to you, because obviously that bid will lose to your new bid if they both stayed active.  If you bid the same amount as your previous bid, it will retain your original bid's timestamp, because, in the event of ties, the bid placed earlier is deemed the winner.  If you place a bid that is less than the minimum bid, those tokens will be immediately returned to you.  Also, if you place a bid after the auction has closed, those tokens will be immediately returned.

The auction will not allow a bid of 0.

It is recommended that the UI designed to send a bid use the optional "padding" field when calling the bid token contract's Send function.  You should make the number of digits of the bid amount + the number of characters in the "padding" field a constant.  That way the size of the Send does not leak information about the size of the bid.  The largest 128-bit unsigned integer has 39 digits, so the number of digits + the number of padding spaces should be at least 40.

## Retract Your Active Bid
You may retract your current active bid with
```sh
secretcli tx compute execute *auction_contract_address* '{"retract_bid": {}}' --from *your_key_alias_or_addr* --gas 300000 -y
```
You may retract your bid at any time before the auction closes to both retract your bid and to return your tokens.  In the unlikely event that your tokens were not returned automatically when the auction ended, you may call retract_bid after the auction closed to return them manually.

## Finalizing the Auction Sale
The auction creator may close an auction with
```sh
secretcli tx compute execute *auction_contract_address* '{"finalize": {"only_if_bids": *true_or_false*}}' --from *your_key_alias_or_addr* --gas 2000000 -y
```
Only the auction creator can finalize an auction.  The boolean only\_if\_bids parameter is used to prevent the auction from closing if there are no active bids.  If there are no active bids, but only\_if\_bids was set to false, then the auction will be closed, and all consigned tokens will be returned to the auction creator. 
If the auction is closed before the auction creator has consigned all the tokens for sale, any tokens consigned will be returned to the auction creator, and any active bids will be returned to the bidders.  If all the sale tokens have been consigned, and there is at least one active bid, the highest bid will be accepted (if tied, the tying bid placed earlier will be accepted).  The auction will then swap the tokens between the auction creator and the highest bidder, and return all the non-winning bids to their respective bidders.

## Returning Funds In The Event Of Error
In the unlikely event of some unforeseen error that results in funds being held by an auction after it has closed, anyone may run
```sh
secretcli tx compute execute *auction_contract_address* '{"return_all": {}}' --from *your_key_alias_or_addr* --gas 2000000 -y
```
Return_all may only be called after an auction is closed.  Auction\_info will indicate whether any funds are still held by a closed auction.  Even if return\_all is not called, bidders who have not received their bids back can still call retract\_bid to have their bids returned.

## View Lists of Active/Closed Auctions
You may view the list of active auctions sorted by pair with
```sh
secretcli q compute query *factory_contract_address* '{"list_active_auctions":{}}'
```
You may view the list of closed auctions in reverse chronological order with
```sh
secretcli q compute query *factory_contract_address* '{"list_closed_auctions":{"before":*optional_u64_timestamp*,"page_size":*optional_u32_number_to_list*}}'
```
If you do not supply the `before` field, it will start with the most recently closed auction, otherwise it will only display auctions that had closed before the provided timestamp.  The timestamp is in the number of seconds since epoch time (01/01/1970).  If you do not supply the `page_size` field, it will default to listing up to 200 closed auctions, otherwise it will display up to the number specifed as `page_size`.

If you are paginating your list, you would take the timestamp of the last auction returned, and specify that in the `before` field of the next query in order to have the subsequent query start where the last one left off.

## View List of Your Auctions
You may view the lists of auctions that you have created, in which you have an active bid, or you have won with
```sh
secretcli q compute query *factory_contract_address* '{"list_my_auctions":{"address":"*address_whose_auctions_to_list*","viewing_key":"*viewing_key*","filter":"*optional choice of active, closed, or all"}}'
```
To view your own auctions, you will need to have created a viewing key with the factory contract.  The `filter` field is an optional field that can be "active", "closed", or "all", to list only active , closed, or all your auctions respectively.  If you do not specify a filter, it will list all your auctions.

## Viewing an Individual Auction's Information
You can view the sell and bid token information, the amount being sold, the minimum bid, the description if present, the auction contract address, and the status of the auction with
```sh
secretcli q compute query *auction_contract_address* '{"auction_info":{}}'
```
Status will either be "Closed" if the auction is over, or it will be "Accepting bids".  If the auction is closed, it will also display the winning bid if there was one.  If the auction is accepting bids, auction_info will also tell you if the auction owner has consigned the tokens to be sold to the auction.  You may want to wait until the owner consigns the tokens before you bid, but there is no risk in doing it earlier.  At any time before the auction closes, you can retract your bid to have your tokens returned to you.  But if you wait until the owner consigns his tokens, you can be more sure the owner is likely to finalize the auction, because once the tokens to be sold are consigned to the auction, he can not get his orignal tokens or the bid tokens until he finalizes the auction.  The original consigned tokens will only be returned if there are no active bids (either no bids were placed meeting the minimum asking price or all qualifying bids have been retracted).  Otherwise, the highest bid placed at the time of closure will be accepted and the swap will take place.

If the auction is closed, it will display if there are any outstanding funds still residing in the auction account.  This should never happen, but if it does for some unforeseen reason, it will remind the user to either use retract\_bid to have their bid tokens returned (if they haven't already been returned), or use return\_all to return all the funds still held by the auction.  Return\_all can only be called after the auction has closed.

## View Your Active Bid in an Individual Auction
You may view your current active bid amount and the time the bid was placed with
```sh
secretcli q compute query *auction_contract_address* '{"view_bid": {"address":"*address_whose_bid_to_list*","viewing_key":"*viewing_key*"}}'
```
You must have created a viewing key with the factory contract before you can view an active bid in an auction.

## Notes for UI builders
It is recommended that the UI designed to send a bid use the optional "padding" field when calling the bid token contract's Send function.  You will want the number of digits of the send amount + the number of characters in the padding field to be a constant number (I use 40 characters, because the maximum number of digits of Uint128 is 39, and I always want at least one blank in padding).  That way the size of the Send does not leak information about the size of the bid.

It would be preferred if the UI immediately executed the consignment of tokens after creating an auction.  A vast majority of users will be using a UI instead of using the secretcli commands.  So if the UIs enforced that they would not create auctions without consigning tokens, then it would be extremely rare for an auction to exist where the sale tokens have not been consigned.  That would just be one less thing the user needs to consider when placing bids.

Don't forget that secret contracts can not use floating points, so any time a token/auction has an amount input/output, it is an integer in the smallest denomination of the token.  Therefore it is up to the UI to convert input/output amounts from/to their decimal equivalent to make things more convenient for the user.  So if your UI accepts a decimal input from the user, it will need to multiply the input by 10^number_of_decimals before creating the execute commands.  That means that when creating a new auction, your UI will need to query the input sell and bid contract addresses to find out how many decimal places they use so that it can translate the sell_amount and minimum_bid to the integer form that the auction and token contracts use.  When placing a bid, if your UI is already pulling the auction_info data when it displays the auction the user wants to bid in, you can just pull the decimal places from the auction_info query instead of needing to query the individual token contracts.  The same applies to responses the UI receives from the auction.  Any amounts will need to be divided by 10^n_decimals if you are going to display amounts in decimal values of whole tokens.  Again, since the UI will likely have already called the auction_info query to display it before the user interacts with the auction, it can store the number of decimals from the auction_info query to use later in translating the response.  When displaying lists of active/closed auctions, the factory will supply the number of decimals (in fields called `sell_decimals` and `bid_decimals`) so that the UI doesn't have to query every auction in the list.

If you are paginating your list of closed auctions, you would take the timestamp of the last auction returned, and specify that in the `before` field of the next query in order to have the subsequent query start where the last one left off.

Also, you should be aware that responses from bidding and consigning (functions that are called indirectly when doing a Send tx with a token contract) are sent in the log attributes.  Also, the address of a newly created auction is returned in a log attribute.  This is because when one contract calls another contract, only logs (not the data field) are forwarded back to the user.  On the other hand, any time you call a contract directly that does not need to call another contract (or that can ignore the other contract's response), the response will be sent in the data field, which is the preferred method of returning json responses.