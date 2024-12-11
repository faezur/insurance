# insurance

// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract Insurance {
    

    struct Policy {
        uint premium;
        uint coverageAmount;
        uint policyDuration;
        uint expirationDate;
        bool isActive;
    }

    struct Claim {
        uint claimAmount;
        string reason;
        bool isApproved;
        bool isPaid;
    }

    address public insurer;
    mapping(address => Policy) public policies;
    mapping(uint => Claim) public claims;
    uint public claimCounter;

    event PolicyIssued(address indexed policyholder, uint premium, uint coverageAmount, uint policyDuration);
    event PremiumPaid(address indexed policyholder, uint amount);
    event ClaimSubmitted(address indexed policyholder, uint claimId, uint claimAmount);
    event ClaimApproved(address indexed policyholder, uint claimId);
    event ClaimPaid(address indexed policyholder, uint claimId, uint amount);

    modifier onlyInsurer() {
        require(msg.sender == insurer);
        _;
    }

    modifier policyActive(address _policyholder) {
        require(policies[_policyholder].isActive, "Policy is not active.");
        require(policies[_policyholder].expirationDate > block.timestamp, "Policy has expired.");
        _;
    }

    constructor() {
        insurer = msg.sender;
    }

    function issuePolicy(address _policyholder, uint _premium, uint _coverageAmount, uint _policyDuration) external onlyInsurer {
        policies[_policyholder] = Policy({
            premium: _premium,
            coverageAmount: _coverageAmount,
            policyDuration: _policyDuration,
            expirationDate: block.timestamp + _policyDuration,
            isActive: true
        });
        emit PolicyIssued(_policyholder, _premium, _coverageAmount, _policyDuration);
    }

    function payPremium() external payable policyActive(msg.sender) {
        require(msg.value == policies[msg.sender].premium, "Incorrect premium amount.");
        emit PremiumPaid(msg.sender, msg.value);
    }

    function submitClaim(uint _claimAmount, string memory _reason) external policyActive(msg.sender) {
        require(_claimAmount <= policies[msg.sender].coverageAmount, "Claim exceeds coverage.");
        claimCounter++;
        claims[claimCounter] = Claim({
            claimAmount: _claimAmount,
            reason: _reason,
            isApproved: false,
            isPaid: false
        });
        emit ClaimSubmitted(msg.sender, claimCounter, _claimAmount);
    }

    function approveClaim(uint _claimId) external onlyInsurer {
        Claim storage claim = claims[_claimId];
        require(!claim.isApproved, "Claim already approved.");
        claim.isApproved = true;
        emit ClaimApproved(msg.sender, _claimId);
    }

    function payClaim(uint _claimId) external onlyInsurer {
        Claim storage claim = claims[_claimId];
        require(claim.isApproved, "Claim not approved.");
        require(!claim.isPaid, "Claim already paid.");
        require(address(this).balance >= claim.claimAmount, "Insufficient contract balance.");
        claim.isPaid = true;
        payable(msg.sender).transfer(claim.claimAmount);
        emit ClaimPaid(msg.sender, _claimId, claim.claimAmount);
    }
}
