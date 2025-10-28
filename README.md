// SPDX-License-Identifier: MIT
pragma solidity ^0.8.0;

contract FreelanceEscrow {
    
    enum ProjectStatus { Created, Funded, Completed, Disputed, Resolved }
    
    struct Project {
        address client;
        address freelancer;
        uint256 amount;
        ProjectStatus status;
        string projectDetails;
        uint256 deadline;
        bool clientApproved;
        bool freelancerCompleted;
    }
    
    mapping(uint256 => Project) public projects;
    uint256 public projectCounter;
    
    event ProjectCreated(uint256 indexed projectId, address indexed client, address indexed freelancer, uint256 amount);
    event ProjectFunded(uint256 indexed projectId, uint256 amount);
    event ProjectCompleted(uint256 indexed projectId);
    event FundsReleased(uint256 indexed projectId, address indexed freelancer, uint256 amount);
    event ProjectDisputed(uint256 indexed projectId);
    event DisputeResolved(uint256 indexed projectId, address winner);
    
    modifier onlyClient(uint256 _projectId) {
        require(msg.sender == projects[_projectId].client, "Only client can call this");
        _;
    }
    
    modifier onlyFreelancer(uint256 _projectId) {
        require(msg.sender == projects[_projectId].freelancer, "Only freelancer can call this");
        _;
    }
    
    modifier projectExists(uint256 _projectId) {
        require(_projectId < projectCounter, "Project does not exist");
        _;
    }
    
    // Core Function 1: Create and Fund Project
    function createProject(
        address _freelancer,
        string memory _projectDetails,
        uint256 _deadline
    ) external payable returns (uint256) {
        require(_freelancer != address(0), "Invalid freelancer address");
        require(_freelancer != msg.sender, "Client and freelancer must be different");
        require(msg.value > 0, "Must send payment");
        require(_deadline > block.timestamp, "Deadline must be in the future");
        
        uint256 projectId = projectCounter;
        
        projects[projectId] = Project({
            client: msg.sender,
            freelancer: _freelancer,
            amount: msg.value,
            status: ProjectStatus.Funded,
            projectDetails: _projectDetails,
            deadline: _deadline,
            clientApproved: false,
            freelancerCompleted: false
        });
        
        projectCounter++;
        
        emit ProjectCreated(projectId, msg.sender, _freelancer, msg.value);
        emit ProjectFunded(projectId, msg.value);
        
        return projectId;
    }
    
    // Core Function 2: Complete Project and Release Funds
    function completeProject(uint256 _projectId) 
        external 
        projectExists(_projectId)
        onlyFreelancer(_projectId) 
    {
        Project storage project = projects[_projectId];
        require(project.status == ProjectStatus.Funded, "Project not in funded state");
        require(block.timestamp <= project.deadline, "Project deadline has passed");
        
        project.freelancerCompleted = true;
        project.status = ProjectStatus.Completed;
        
        emit ProjectCompleted(_projectId);
    }
    
    function approveAndReleaseFunds(uint256 _projectId)
        external
        projectExists(_projectId)
        onlyClient(_projectId)
    {
        Project storage project = projects[_projectId];
        require(project.status == ProjectStatus.Completed, "Project not completed");
        require(project.freelancerCompleted, "Freelancer has not marked as completed");
        
        project.clientApproved = true;
        project.status = ProjectStatus.Resolved;
        
        uint256 amount = project.amount;
        project.amount = 0;
        
        payable(project.freelancer).transfer(amount);
        
        emit FundsReleased(_projectId, project.freelancer, amount);
    }
    
    // Core Function 3: Dispute Resolution
    function raiseDispute(uint256 _projectId)
        external
        projectExists(_projectId)
    {
        Project storage project = projects[_projectId];
        require(
            msg.sender == project.client || msg.sender == project.freelancer,
            "Only client or freelancer can raise dispute"
        );
        require(
            project.status == ProjectStatus.Funded || project.status == ProjectStatus.Completed,
            "Cannot dispute in current state"
        );
        
        project.status = ProjectStatus.Disputed;
        
        emit ProjectDisputed(_projectId);
    }
    
    function resolveDispute(uint256 _projectId, bool _favorFreelancer)
        external
        projectExists(_projectId)
        onlyClient(_projectId)
    {
        Project storage project = projects[_projectId];
        require(project.status == ProjectStatus.Disputed, "Project not in dispute");
        
        project.status = ProjectStatus.Resolved;
        uint256 amount = project.amount;
        project.amount = 0;
        
        if (_favorFreelancer) {
            payable(project.freelancer).transfer(amount);
            emit DisputeResolved(_projectId, project.freelancer);
        } else {
            payable(project.client).transfer(amount);
            emit DisputeResolved(_projectId, project.client);
        }
    }
    
    // View Functions
    function getProjectDetails(uint256 _projectId)
        external
        view
        projectExists(_projectId)
        returns (
            address client,
            address freelancer,
            uint256 amount,
            ProjectStatus status,
            string memory projectDetails,
            uint256 deadline
        )
    {
        Project memory project = projects[_projectId];
        return (
            project.client,
            project.freelancer,
            project.amount,
            project.status,
            project.projectDetails,
            project.deadline
        );
    }
    
    function getContractBalance() external view returns (uint256) {
        return address(this).balance;
    }
}



0xa8cf3d9Fcc4C4b5b22b045C7855732ccF3f348d0

<img width="1584" height="713" alt="image" src="https://github.com/user-attachments/assets/19271975-62fa-4acc-aa19-a5079b58216b" />
