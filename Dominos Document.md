# Dominos  Design Document

## Gameplay Design

1. Highest Drawer plays first tile (2 Moves Technically)
2. Each player places one domino per hand (North, South, West, East)
3. If player cannot make a move they must take a tile from the Train Yard, this can be played or not, otherwise forfeit Move to next player.
4. Repeat Steps 2-3 until all player have no more valid moves or player has no more Dominios is the Win State.
5. Calculate Score based on End State Lowest Score Wins, draws can happen.

Download: [Dominos Game Design Diagram](https://github.com/staticJPL/SGIDomino/blob/39a7dfc528b2d0028dd70685c97fa232b168f082/Gameplay%20Design.svg)
## Data Layout, Sample Code

### DominoBase

**Purpose**: Base class for domino tiles, containing basic properties and methods for all dominoes.

```cpp
struct DominoBase
{
	int SideA;
	int SideB;
	DominoBase Parent = nullptr; // Thing it's attached too
	DominoBase Child  = nullptr; // Check if it has a open spot.
	DominioBase(int sa,int sb) : sideA(sa),sideB(sb){}
	// Additonal Operators if needed
	virtual ~DominosBase(){}
}
```

###  DominoRoot

 **Purpose**: Represents a tile in the linked list of the game board, with connections to other tiles. Used for Polymorphic cast and STD::move data into a RootNode object.
 
```cpp
// Derived struct for root-specific functionality
struct RootDomino : public DominoBase {

	// Should be unique pointers (Smart pointers)
    DominoBase* north;
    DominoBase* south;
    DominoBase* west;
    DominoBase* east;

    // Constructor for root domino, calls base struct constructor
    RootDomino(int s1, int s2)
        : DominoBase(s1, s2), north(nullptr), south(nullptr), west(nullptr), east(nullptr) {}

    // Move constructor steal resources from initial tile
    RootDomino(RootDomino&& other) noexcept
        : DominoBase(other.side1, other.side2), north(other.north), south(other.south),
          west(other.west), east(other.east) {
        // Nullify the pointers in the moved-from object
        other.north = nullptr;
        other.south = nullptr;
        other.west = nullptr;
        other.east = nullptr;
    }

    // Move assignment operator
    RootDomino& operator=(RootDomino&& other) noexcept {
        if (this != &other) {
            // Move base class data
            DominoBase::operator=(std::move(other));
            north = other.north;
            south = other.south;
            west = other.west;
            east = other.east;

            // Nullify the pointers in the moved-from object
            other.north = nullptr;
            other.south = nullptr;
            other.west = nullptr;
            other.east = nullptr;
        }
        return *this;
    }
};
```
### TrainPool

**Purpose**: Manages extra dominoes that remain after splitting Tiles equally. Used for players to Draw from when no moves are available.

```cpp
// Pool Remainder will always be ( (Mtiles - Nplayer ) % Nplayers)
// If this scales with more players or tiles*

struct TrainPool {
	std::vector<std::unique_ptr<DominoBase>> extraTiles; // Holds extra dominoes // Function to add a tile to the Train Pool 
	void AddTile(std::unique_ptr<DominoBase> tile); // Function to draw a tile from the Train Pool 
	std::unique_ptr<DominoBase> DrawTile(); 
};
```
### MoveCMD

**Purpose**: Holds simple client move command sent to server, sends the index for the players tile, the direction meaning where to append on the linked list of the game board.

```cpp
enum class Direction{
	None, // 0
	North, // 1
	South, // 2
	East, // 3
	West, // 4
};

struct MoveCmd
{
	size_t playerTileIndex; // Array Index of the player's tile
	Direction direction; // Direction to attach the tile (North, South, East, West)
	Serialize() const{
	... // Serialize MoveCmd For network
	}
}
```

### GameBoard

**Purpose**: Class that holds the start of the Linked List Root Domino. Pointer Offsets are cached for each board update on client/server (North,East,West,South). Also handles if the move is valid and can be attached if numbers match.

```cpp
[Replicated] Server/Client
class GameBoard {
public:
    std::unique_ptr<DominoRoot> root;
    
	size_t NorthOffset; // Offset for End of LinkedList
	size_t SouthOffset;
	size_t EastOffset;
	size_t WestOffset
  // The central starting point of the board and is unique

    // Constructor to initialize an empty board with no root tile
    GameBoard() : root(nullptr) {}

	void SetRoot(DominoRoot& RootTile){
		root = std::Move(RootTile) // Transferring ownership and setting gameboard root. This would be a allocate new in Java... 
	}

    // Function to attach a new tile to an existing one in a given direction
    // Returns true if Valid Move and sends NewMove to Server
    bool attachTile(MoveCmd& NewMove) {
	     // CheckIfTurn From Server
		 // Read MoveCmd PlayerIndex and Direction (North,South,East,West)
		// Append to Direction
		// NorthOffset++,SouthOffset++, EastOffset++,WestOffset++
		// Compare side, if valid return True else return false
		return true;
	}
	void UpdateBoard(MoveCmd& NewMove){
		// Logic here to Update Gameboard pointers and GameboardState
	}

```
### LocalPlayer

**Purpose**: Local Player Class is the actual client object which acts as the wrapper to update the player state with respect to the Server Game State.

```cpp
// Local Player Client
class PlayerActor {
public:
    // Tile Selected by Player from the pool
    void addTile(std::unique_ptr<DominoBase> tile) {
        tiles.push_back(std::move(tile));
    }

    // Find a tile by its values
    DominoBase* findTile(int side1, int side2) {
        for (const auto& tile : tiles) {
            if (tile->side1 == side1 && tile->side2 == side2) {
                return tile.get();
            }
        }
        return nullptr;
    }

	bool isTurn() const{
	
		return PlayerState->GetTurn();
	}

	void AttachToBoard(size_t PlayerTileIndex,Direction CurrentDirection)
	{
		Gameboard* GameboardState = GetGameBoard();
		// Create Command
		MoveCmd newCommand = {PlayerTileIndex,Direction}
		bool isMoveValid = GameboardState->AttachTile(NewCommand);
		
		if(isMoveValid){
			RemoveTile(PlayerTileIndex);
			PlayerState->SendMove(NewCommand); // Send to Server to Update
		}
	}
	
private:

    std::vector<std::unique_ptr<DominoBase>> tiles;
    
	int CurrentScore;
	
	// Helper function to calculate the total score 
	int calculateTotalScore() const
	{ 
		int total = 0; 
		for (const auto& tile : tiles) 
		{ 
			total += tile->side1 + tile->side2; 
		} 
		return total; 
	}
}
```

### PlayerState

**Purpose**: Represents the Player state being replicated and synchronized with the server, tiles, game board and score.

```cpp

// Player State Data is what is checked and Synchronized
class PlayerState {
public:
    PlayerState() 
        : currentScore(0) 
    {}

    // Add a tile to the player's collection and update the score
    void addTile(std::unique_ptr<DominoBase> tile) {
        currentScore += tile->side1 + tile->side2;
        tiles.push_back(std::move(tile));
    }

    // Remove a tile by index from the player's collection and update the score
    void removeTile(size_t index) {
        if (index < tiles.size()) {
            currentScore -= tiles[index]->side1 + tiles[index]->side2;
            tiles.erase(tiles.begin() + index);
        }
    }

    // Serialize the player's state to Send Over Network
    std::string serialize() const {
        std::ostringstream oss;
        oss << currentScore << "\n";
        for (const auto& tile : tiles) {
            oss << tile->serialize() << "\n";
        }
        return oss.str();
    }
	[Replicated RPC Server Only]
	inline void setTurn(bool CurrentTurn){
		bisTurn = CurrentTurn; 
	}
	inline void GetTurn() const{
		return bTurn;
	}
	[Replicated RPC Client->Server]
	void sendMove(MoveCmd& NewMove){
		// Sends Attach Movement to Server
		CallServerRPC(NewMove);
	}
	[Replicated RPC] Server->Client
	void RecievedMove(MoveCmm* ServerMove){
	
		GameBoard* CurrentGameBoardState = GetGameBoard();
		CurrentGameBoardState->UpdateBoard(ServerMove);
	}
	
	Gameboard* GetGameBoard() const{
	
		return GameBoardState;
	}
	PlayerActor* GetPlayer() const{
		return thisPlayer;
	}

private:
	[Replicated]
	std::vector<std::unique_ptr<DominoBase>> tiles;
	
	[Replicated]
	int currentScore;
	
	[Replicated]
	bool bisTurn = false;
	
	[Replicated] Server / Client 
	GameBoard* GameBoardState;
	
	[Replicated] Server / Client
	TrainPool* TilePool;
	
	// Actual Player this player state belongs too
	PlayerActor* thisPlayer;
};

```
### GameMode

**Purpose**: Server Only Manages the Game State for all players, This class defines the game rules, a version of the game board and winning conditions. 

```cpp

// This is a singlton class (Server Authority) means server only
class GameMode {
public:

	// Constructor
	GameMode(size_t numPlayers);
	
	// Static method to get the singleton instance 
	inline static GameMode& GetInstance() { static GameMode instance;}
	
	// Delete copy constructor and assignment operator so you cannot make copies or make a non copyable class to inheirt from.
	GameMode(const GameMode&) = delete; 
	GameMode& operator=(const GameMode&) = delete;
	
	// Function to generate a random set of dominoes
	// Send Seeded set to players
	void GenerateRandomDominoSet_RPC();
	
	// Function to check winning conditions
	bool CheckWinningConditions() const;
	
	// Update Gameboard based on PlayerStates
	void UpdateGameBoard();
	
	// Send Updates to connected Clients
	void NotifyPlayersBoardUpdate()
	
	// Function to get the current player
	PlayerState* GetCurrentPlayer() const;
	
	// Function to switch to the next player
	// Set bTurn on Client
	void NextTurn_RPC();
	
	// Function to get the number of players
	size_t GetNumPlayers() const;
	
	// Function to get the current game board
	const GameBoard& GetGameBoard() const;
	
	// Function to get the Train Pool
	const TrainPool& GetTrainPool() const;

private:
	std::vector<std::unique_ptr<PlayerState>> playerStates; // List of player states
	GameBoard gameBoard; // Game board Server Version
	TrainPool trainPool; // Train Pool for extra tiles
	size_t currentPlayerIndex; // Index of the current player
	size_t numPlayers; // Number of players
	};

```

## System Design

#### Game Engine 

1. **Custom Game Engine**

	Requirements:
		Languages: C++, Java, C#, JavaScript
		Networking: ENet,RakNet
		Rendering: Vulkan, OpenGL, DirectX12, WebGL,Three.js
	Pros:
	- Simplicity
	- Customizability
	- Lean/Minimal Code base

	Cons:
	- High learning curve
	- Upfront development time
	- Limited Features
	- Reinventing systems
	- Lack of debugging tools

2. **Unreal/Unity/Godot Engine**

	Requirements:
		Languages: C++, C#
		Networking: Provided
		Rendering: Provided
	Pros:
	- Faster Prototyping or Development
	- Cross platform support
	- Out of the box rendering, network support
	- Large Community or outreach
	- Turn key to potential solutions

	Cons:
	- Potentially High Learning Curve
	- Less flexibility or customizability
	- Large code base
	- Performance overhead
	- Heavy complexity and layered abstractions


**Choice:** Unreal Engine

Reason: Built in Networking, Rendering and Cross Platform Support.

### System Design

#### Network Architecture

1. **Peer to Peer** (P2P)

	Pros:
	- Cheaper 
	- Decentralized
	- Self distributed scaling

	Cons:
	- Security Vulnerabilities Risks of Cheating
	- No Central Control
	- Inconsistent Networking bandwidth, routing, performance
	- Open to Network Attacks DDOS

2. **Client-Server** 

	Pros:
	- Cheaper depending on use case
	- Decentralized
	- Self distributed scaling

	Cons:
	- Security Vulnerabilities Risks of Cheating
	- No Central Control
	- Inconsistent Networking bandwidth, routing, performance
	- Open to Network Attacks (DDos etc..)

	2a. **Self Hosted (Bare metal)** 
	
	Pros:
	- Full Control (Data Ownership, Security)
	- Long term cost saving from vendor price gouging depending of life of project
	- Offline Availability, local network access etc.
	
	Cons:
	- Scalability issues
	- Potentially high input costs
	- Long term maintenance
	- IT expertise difficult to find
	- Data loss risks
	- Potential for Local Data breaches
	  
	2b. **Cloud Services**
	
	Pros: 
	- Fast Deployment
	- No hardware requirements
	- Dynamic Scaling (Auto Scaling)
	- High Availability with redundancy 
	- Security and Compliance(DDos Protection etc..)
	- Robust management tools (Deployment, Microservices)
	Cons:
	- Higher Cost, unpredictable hidden fees
	- Still not immune to data breaches
	- Less flexibility over infrastructure, vanilla cake, chocolate cake setups
	- Vendor dependency, jumping through hoops to fix simplistic things.

**Choice:** Cloud Dedicated server

Reason: Used in most cases, Dedicated servers can also be self hosted. Thus, if scalability is expected then it can be deployed with a Cloud Service.

### Internal Architecture

**Source Control**
	- Git
	- Perforce
	- Subversion

Choice: Git or Perforce, usually perforce since it handles Binaries better. Git can also be used along side. 

Bare metal hosting for services needed for `Testing`, `Version Control`.

Source Control sever contains repositories for Development/Testing respectively. 

### Development

Unreal Engine generate two modules `Dominio` and `Domino Server`. Both are packaged differently where the `Domino Server` is headless version since rendering is not needed.

After both modules are generated a `Unit Test Module` performs a **smoke test** to pass packaged builds to `Test Environment`. 
### Testing

Testing is performed for each Module separately

Some testing examples

- Performance testing
- Gameplay testing
- UX testing
- Regression Testing

### Staging/Integration

Mirrors the Production Server environment. Testing is performed with various different test machines of different hardware configurations. In addition any networking is pen tested before transitioning to Production. Modules go through a `Integration Test` to iron out issues within the cloud environment.

### Production

After Passing Staging and Integration the Modules are deployed to Production.

The production environment will have typical cloud services that manage the creation of Game Servers, Software distribution servers (File server) while receiving updates from Staging/Integration.

Furthermore a Red/Blue redundancy in the production environment allows for seamless swapping after any module patches (Sever or Client). This can happen if any game breaking bugs are caught during Testing/Staging. If a swap occurs the system repoints client seamlessly to the Red or Blue server layer without knowing it. 

### External Access (Contractor, Remote Worker)

Connection to Internal services (Source Control, Build Environment) will be externally proxied with proper security protocols (SSL,HTTPS etc..). An external porta is used to login and tunnel behind the Local Firewall. Extra Layer of User Authentication checks if users connected are valid then forwards Network traffic to the Proxy Subnet containing all the Internal services used by the company (Development,Test, Version Control). Any baremetal hardware creating the Virtual Machines should not visible, this can be done in many way virutal networks, another reverse proxy etc..

### System Design Diagram
Download: [Dominos Game SGI/System Design](https://github.com/staticJPL/SGIDomino/blob/39a7dfc528b2d0028dd70685c97fa232b168f082/System%20Design.svg)
