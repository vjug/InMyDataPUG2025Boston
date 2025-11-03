# Step 5: Introducing the AI Agent (LiveKit Integration)

## What Changed in This Step

This step introduces the AI agent using LiveKit and OpenAI, transforming the database driver into an interactive voice-enabled assistant that can look up and create car records through natural conversation.

### New/Modified Files

- **`requirements.txt`**: Major expansion with AI/voice dependencies:
  - `python-dotenv`, `requests` (from previous steps)
  - **`livekit-agents[openai,silero,turn-detector]`**: Core LiveKit agents framework
  - **`livekit-plugins-openai`**: OpenAI integration for the LLM
  - **`livekit-plugins-silero`**: Voice activity detection
  - **`livekit-api`**: LiveKit API client

- **`agent.py`**: **NEW** - Defines the AI agent class:
  - `Assistant` class inheriting from LiveKit's `Agent`
  - Function tools for car lookup and creation
  - Integration with `OEDatabaseDriver`

- **`prompts.py`**: **NEW** - Contains agent instructions and welcome messages:
  - `WELCOME_MESSAGE`: What the agent says when the user connects
  - `INSTRUCTIONS`: System prompt defining agent behavior

- **`main.py`**: **NEW** - Agent entry point:
  - Sets up the LiveKit session
  - Configures OpenAI Realtime model (voice)
  - Starts the agent and connects to the room

- **`OEDatabaseDriver.py`**: Minor updates to support agent integration

- **`.env`**: Updated to include:
  - `LIVEKIT_URL`: Your LiveKit server URL
  - `LIVEKIT_API_KEY`: API key for LiveKit
  - `LIVEKIT_API_SECRET`: API secret for LiveKit
  - `OPENAI_API_KEY`: OpenAI API key

### Code Overview

#### Agent Class (agent.py)

```python
class Assistant(Agent):
    def __init__(self) -> None:
        super().__init__(instructions=INSTRUCTIONS)
        self._car_details = Car()
    
    @function_tool
    async def lookup_car_by_registration_number_in_database(
        self, 
        reg: Annotated[str, "Car registration number"]
    ):
        result = driver.get_car(reg.upper().replace(" ", ""))
        if result is None:
            return "Car not found"
        self._car_details = result
        return f"The car details are: {self.get_car_str()}"
    
    @function_tool
    async def add_car_details_to_database(
        self, 
        reg: Annotated[str, "The registration number (reg) of the car"],
        make: Annotated[str, "The make of the car"],
        model: Annotated[str, "The model of the car"],
        year: Annotated[int, "The year of the car"]
    ):
        result = driver.save_car(reg, make, model, year)
        self._car_details = Car(reg=reg, make=make, model=model, year=year)
        return "car created!"
```

#### Agent Instructions (prompts.py)

```python
INSTRUCTIONS = """
    You are the manager of a UK call center, speaking to a customer. 
    Start by asking for the customer's car registration number (reg).
    If found in database, answer their questions.
    If not found, create an account by asking for make, model, and year.
"""
```

#### Entry Point (main.py)

```python
async def entrypoint(ctx: agents.JobContext):
    session = AgentSession(
        llm=openai.realtime.RealtimeModel(
            voice="shimmer",
            temperature=0.8,
        )
    )
    
    await session.start(room=ctx.room, agent=Assistant())
    await ctx.connect()
    await session.generate_reply(instructions=WELCOME_MESSAGE)
```

## What This Implements

**AI-Powered Voice Agent**: This step brings together multiple technologies:

- **Natural Language Understanding**: OpenAI's GPT model understands user intent from conversational speech
- **Voice Interaction**: LiveKit's Realtime API enables natural voice conversations
- **Function Calling**: The LLM can decide when to call database functions based on conversation context
- **State Management**: The agent tracks the current car details across the conversation
- **Structured Prompts**: System instructions guide the agent's behavior and personality

Key capabilities:
- User speaks: "My registration is GJ24 YBR"
- Agent calls `lookup_car_by_registration_number_in_database("GJ24YBR")`
- Agent responds: "I found your car - it's a 2018 Audi A4"

## How This Helps Build Towards the Final Solution

This is a transformative step that establishes the core agent architecture:

1. **Conversational Interface**: Users can now interact naturally instead of using forms or command-line tools

2. **Function Tools Pattern**: The `@function_tool` decorator pattern will be used throughout:
   - Each Python method becomes a tool the LLM can invoke
   - The LLM decides WHEN to call tools based on conversation
   - Parameters are extracted from natural language

3. **Agent Architecture**: Establishes the pattern:
   ```
   User Speech → LiveKit → OpenAI → Function Call → OEDatabaseDriver → Database
                     ↑                                                      ↓
                     ←──────── Speech Response ←─── Agent ←──── Result ←───
   ```

4. **Stateful Conversations**: The agent maintains context:
   - Remembers which car is being discussed
   - Can reference previous information
   - Provides coherent multi-turn conversations

5. **Development Mode**: The `python main.py dev` command connects to the LiveKit Agents Playground for easy testing

## Testing This Step

1. Ensure both services are running:
   - OpenEdge PASOE with car service
   - Database with some test car records

2. Start the agent in development mode:
   ```powershell
   py main.py dev
   ```

3. Open the [LiveKit Agents Playground](https://agents-playground.livekit.io/)

4. Connect to your agent using your LiveKit credentials

5. **Test Scenarios**:

   **Scenario 1: Lookup Existing Car**
   - Say: "Hi, my registration is GJ24 YBR"
   - Agent should find the car and read back the details

   **Scenario 2: Create New Car**
   - Say: "My registration is XYZ 123"
   - Agent: "Car not found, I'll create an account"
   - Agent asks for make, model, year
   - You provide: "It's a 2020 Toyota Corolla"
   - Agent creates the car record

6. **Verify in Database**: Use `OpenEdge\ViewCars.w` to confirm records were created

## Key Concepts for OpenEdge Developers

- **Agent ≈ Persistent Procedure**: The `Assistant` class maintains state across interactions, like a persistent procedure that stays loaded

- **Function Tools ≈ Published Methods**: The `@function_tool` decorator is like marking methods as available for external calling:
  ```abl
  METHOD PUBLIC CHARACTER lookupCar(INPUT cReg AS CHARACTER):
      /* implementation */
  END METHOD.
  ```

- **Async/Await**: Python's `async`/`await` is for handling concurrent operations - the agent can handle multiple conversations simultaneously

- **LLM as Orchestrator**: The OpenAI model acts like a smart dispatcher that:
  - Understands user intent
  - Decides which function to call
  - Formats the results for the user
  - All without hard-coded if/then logic!

## What's Next

Step 6 will add booking functionality, allowing the agent to schedule service appointments in addition to managing car records.
