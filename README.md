# CA3-Blockchain-Arch.-and-Design-II

**Question No. Assigned: 3**

**Original Code :**


function withdraw() external {
     
     uint256 amount = balances[msg.sender];
    
    (bool success,) = msg.sender.call{value: balances[msg.sender]}("");
    
    require(success);
    
    balances[msg.sender] = 0;
}

Imagine you are doing a manual audit and you come across above code. Write a comprehensive report explaining the issue and the fix for the issue.

**Answer:-**

**Audit Report: Vulnerability in withdraw Function**

**Description of the Function:**

The withdraw function allows users to withdraw their balance stored in the balances mapping. It uses msg.sender.call to send Ether and updates the user’s balance in the balances mapping to 0 after performing the transfer.

________________________________________

**Identified Issue: Reentrancy Vulnerability**

The function contains a reentrancy vulnerability, which arises due to the order of operations in the function:
1.	The balance of the user (balances[msg.sender]) is retrieved and used for transferring Ether using msg.sender.call.
2.	The balance in the balances mapping is reset to 0 only after the transfer is completed.
If msg.sender is a contract with malicious logic in its fallback or receive function, it can exploit this order of operations by recursively calling the withdraw function before the balance is set to 0. This allows the attacker to withdraw their balance multiple times, effectively draining the contract’s Ether.

________________________________________

**Severity:**

•	Critical: Exploitation of this vulnerability can result in the complete loss of funds held by the contract.

________________________________________

**Exploit Scenario:**

1.	Assume the attacker’s msg.sender balance is 10 Ether.
2.	The attacker calls withdraw, which triggers the transfer using msg.sender.call.
3.	During the transfer, the attacker’s contract executes its fallback function and recursively calls withdraw before the balances[msg.sender] is set to 0.
4.	Each recursive call will withdraw 10 Ether again and again, as the balance remains unchanged until the end of the execution.
5.	The attacker can continue this process until the contract's Ether balance is depleted.
   
________________________________________

**Recommended Fix:**

To prevent reentrancy, the function should follow the checks-effects-interactions pattern:
1.	Update the state variables before making any external calls.
2.	Minimize the use of low-level call, and use transfer or send wherever possible. If call is necessary, handle its return value carefully.
   
________________________________________

**Revised Code:**

function withdraw() external {
    uint256 amount = balances[msg.sender];
    require(amount > 0, "Insufficient balance");

    // **Update state before interaction**
    balances[msg.sender] = 0;

    // **Transfer funds**
    (bool success, ) = msg.sender.call{value: amount}("");
    require(success, "Transfer failed");
}

________________________________________
**Explanation of Fix:**

**1.	Order of Operations:**

	The user’s balance in the balances mapping is set to 0 before making the external call. This prevents any recursive calls from accessing the previous balance.
 
**2.	Reentrancy Mitigation:**
   
	By resetting the balance before interacting with msg.sender, any recursive call would see the balance as 0, preventing further withdrawals.

**3.	Additional Safety:**

	Added a check (require(amount > 0)) to **ensure the user has a non-zero balance before proceeding.**
	Retained the require(success) check to **ensure the transfer completes successfully.**
________________________________________

**Additional Recommendations:**

1.	Use ReentrancyGuard: Consider using OpenZeppelin's ReentrancyGuard to add a layer of protection against reentrancy attacks:

import "@openzeppelin/contracts/security/ReentrancyGuard.sol";

contract MyContract is ReentrancyGuard {
    function withdraw() external nonReentrant {
        uint256 amount = balances[msg.sender];
        require(amount > 0, "Insufficient balance");
        balances[msg.sender] = 0;
        (bool success, ) = msg.sender.call{value: amount}("");
        require(success, "Transfer failed");
    }
}

2.	Audit the Use of call:
   
	Low-level call can introduce additional risks if not handled properly. Consider using transfer or send for transferring Ether where possible. However, note that transfer has a 2300 gas stipend limit which might not be suitable in all cases.

3.	Implement Regular code audits
________________________________________

**Conclusion:**

The original implementation is critically vulnerable to reentrancy attacks. By following the checks-effects-interactions pattern and/or using a reentrancy guard, the vulnerability can be mitigated, ensuring the contract's funds remain secure.
