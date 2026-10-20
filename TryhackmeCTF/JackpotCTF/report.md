# Penetration Testing Report: Jackpot CTF

**Target IP:** 10.114.144.230  
**Challenge Name:** Jackpot (TryHackMe)  

## 1. Executive Summary
An assessment was performed against the Jackpot ATM terminal simulation. Full control over the cash dispensing unit was achieved through XFS bridge protocol exploitation, resulting in the successful draining of all cassettes and retrieval of the challenge flag.

## 2. Exploitation Summary
- **Nonce:** 531505
- **Activation Code:** 03865900
- **Captured Flag:** THM{n0_card_n0_pin_just_the_wr0ng_v0ice_speaking_XFS}
