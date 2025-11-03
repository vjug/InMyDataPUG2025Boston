# Step 7: Multi-Agent Architecture

## What Changed in This Step

This step refactors the single-agent system into a multi-agent architecture, splitting responsibilities between specialized agents with automatic handoff. This demonstrates advanced agent orchestration and separation of concerns.

### New/Modified Files

- **`requirements.txt`**: No changes (uses same dependencies as Step 6)

- **`accountAgent.py`**: **NEW** - First agent in the workflow:
  - Inherits from `Agent`
  - Handles car lookup and registration
  - Transfers control to `BookingAssistant` after finding/creating car
  - Uses "shimmer" voice

- **`bookingAgent.py`**: **NEW** - Second agent in the workflow:
  - Inherits from `Agent`
  - Handles all booking operations
  - Receives car details from `AccountAssistant`
  - Uses "echo" voice
  - Implements `on_enter()` lifecycle method

- **`prompts.py`**: Enhanced with agent-specific instructions:
  - `ACCOUNT_INSTRUCTIONS`: Guides the account agent behavior
  - `BOOKING_INSTRUCTIONS`: Guides the booking agent behavior
  - `WELCOME_MESSAGE`: Initial greeting from account agent

- **`main.py`**: Modified to start with `AccountAssistant`:
  - Uses `AgentSession[Car]` for typed state passing
  - Starts the account agent by default

- **`agent.py`**: **REMOVED** - Replaced by specialized agents

- **`OEDatabaseDriver.py`**: No changes (same as Step 6)

### Code Overview

#### Account Agent (accountAgent.py)

```python
class AccountAssistant(Agent):
    def __init__(self) -> None:
        super().__init__(
            instructions=ACCOUNT_INSTRUCTIONS,
            llm=openai.realtime.RealtimeModel(
                voice="shimmer",  # Different voice for this agent
                temperature=0.8
            )
        )
        self.car = Car()
    
    @function_tool
    async def lookup_car_by_registration_number_in_database(
        self, 
        reg: Annotated[str, "Car registration number"]
    ):
        result = driver.get_car(reg.upper().replace(" ", ""))
        if result is None:
            return "Car not found"
        
        self.car = result
        # HANDOFF: Return the booking agent and transfer message
        return BookingAssistant(car=self.car), "Transfer to booking agent"
    
    @function_tool
    async def add_car_details_to_database(
        self, 
        reg: Annotated[str, "The registration number (reg) of the car"],
        make: Annotated[str, "The make of the car"],
        model: Annotated[str, "The model of the car"],
        year: Annotated[int, "The year of the car"]
    ):
        result = driver.save_car(reg, make, model, year)
        self.car = Car(reg=reg, make=make, model=model, year=year)
        # HANDOFF: Return the booking agent and transfer message
        return BookingAssistant(car=self.car), "Transfer to booking agent"
```

#### Booking Agent (bookingAgent.py)

```python
class BookingAssistant(Agent):
    def __init__(self, car: Car) -> None:
        self.car = car  # Receive car from AccountAssistant
        super().__init__(
            instructions=BOOKING_INSTRUCTIONS,
            llm=openai.realtime.RealtimeModel(
                voice="echo",  # Different voice for this agent
                temperature=0.8
            )
        )
    
    async def on_enter(self) -> None:
        """Called when this agent becomes active"""
        booking = driver.get_booking(self.car.reg)
        if booking is not None:
            # User has existing booking
            await self.session.generate_reply(
                instructions=f"""Tell the customer their car details:
                    Registration: {self.car.reg}, 
                    Make: {self.car.make},
                    Model: {self.car.model},
                    Year: {self.car.year}.
                Also tell them about their existing booking on {booking.booking_date}."""
            )
        else:
            # No existing booking
            next_date = driver.get_next_available_booking(date.today())
            await self.session.generate_reply(
                instructions=f"""Tell customer their car details and that 
                they have no existing bookings. Next available date is {next_date}.
                Ask if they'd like to make a booking."""
            )
    
    @function_tool
    async def get_car_details(self):
        return f"The car details are: {self.get_car_str()}"
    
    # ... booking tools (get_the_date_today, book_appointment, etc.)
```

#### Updated Entry Point (main.py)

```python
async def entrypoint(ctx: agents.JobContext):
    session = AgentSession[Car]()  # Type parameter for state passing
    
    await session.start(
        room=ctx.room,
        agent=AccountAssistant()  # Start with account agent
    )
    
    await ctx.connect()
    await session.generate_reply(instructions=WELCOME_MESSAGE)
```

## What This Implements

**Multi-Agent Orchestration**: This step introduces several advanced patterns:

- **Agent Specialization**: Each agent has a focused responsibility:
  - `AccountAssistant`: Account management (car lookup/creation only)
  - `BookingAssistant`: Booking management (appointments only)

- **Automatic Handoff**: When the account agent finds or creates a car, it returns a tuple:
  ```python
  return BookingAssistant(car=self.car), "Transfer to booking agent"
  ```
  LiveKit automatically switches to the returned agent.

- **State Passing**: The car details are passed from one agent to another via constructor parameters

- **Lifecycle Methods**: The `on_enter()` method runs when an agent becomes active, allowing it to:
  - Perform initialization
  - Check database state
  - Generate a contextual greeting

- **Voice Differentiation**: Each agent uses a different voice ("shimmer" vs "echo"), helping users understand which agent they're talking to

## How This Helps Build Towards the Final Solution

This architectural pattern provides significant benefits:

1. **Separation of Concerns**: Each agent has a single, well-defined responsibility:
   - Easier to maintain
   - Easier to test
   - Clearer instructions/prompts
   - Less confusion for the LLM

2. **Scalability**: Easy to add more specialized agents:
   - PaymentAgent for processing payments
   - InventoryAgent for parts availability
   - SupportAgent for technical questions
   - Each can have different voices, personalities, and tools

3. **Context Management**: The handoff pattern ensures:
   - No loss of information between agents
   - Type-safe state passing (using `AgentSession[Car]`)
   - Clear ownership of data

4. **Better User Experience**: 
   - Voice changes signal agent switches
   - Contextual greetings (agent knows if you have bookings)
   - Smoother conversations with appropriate agent for each task

5. **Professional Architecture**: Mirrors real call center operations:
   - Account verification team → Booking team
   - Each team has specialized training (different instructions)
   - Automated routing (handoff logic)

6. **Reduced Prompt Complexity**: Instead of one long prompt with all instructions:
   ```
   Before: "You handle accounts AND bookings AND payments AND..."
   After:  "You ONLY handle account lookup and creation, then transfer"
   ```
   Simpler prompts lead to more reliable behavior.

## Testing This Step

Start the agent:
```powershell
py main.py dev
```

Connect via [LiveKit Agents Playground](https://agents-playground.livekit.io/)

### Test Complete Flow

**Scenario 1: Existing Customer with Booking**
1. You: "Hi, my registration is GJ24 YBR"
2. AccountAgent: [Looks up car] "Found your car"
3. **[HANDOFF - Voice changes from shimmer to echo]**
4. BookingAgent: "You have a 2018 Audi A4. You have an existing booking on [date] for [description]"
5. You: "Can I reschedule to next week?"
6. BookingAgent: [Handles booking changes]

**Scenario 2: New Customer, No Booking**
1. You: "My reg is NEW 123"
2. AccountAgent: "Not found, I'll create an account. What's the make, model, and year?"
3. You: "2021 Honda Civic"
4. AccountAgent: [Creates car record]
5. **[HANDOFF - Voice changes]**
6. BookingAgent: "You have a 2021 Honda Civic. No bookings yet. Next available is [date]. Would you like to book?"
7. You: "Yes, for a service"
8. BookingAgent: [Books appointment]

### What to Observe

- **Voice Change**: Listen for the voice changing from "shimmer" to "echo" during handoff
- **Context Retention**: The booking agent knows your car details without asking again
- **Proactive Information**: The booking agent checks for existing bookings automatically
- **Focused Conversations**: Account agent doesn't talk about bookings; booking agent doesn't create accounts

## Key Concepts for OpenEdge Developers

- **Multi-Agent ≈ Multi-Procedure Architecture**: Like having separate procedures for different responsibilities:
  ```abl
  RUN accountManagement.p.
  /* Based on result, run appropriate next procedure */
  RUN bookingManagement.p (INPUT hCar).
  ```

- **Handoff ≈ RUN with Parameters**: 
  ```abl
  RUN bookingAgent.p (INPUT TABLE ttCar).
  ```
  The car details are passed as parameters to the next agent.

- **on_enter() ≈ Persistent Procedure ENTRY**: Like initialization code that runs when a persistent procedure is first invoked

- **Agent Session ≈ Shared Memory**: The `AgentSession[Car]` is like a shared temp-table or buffer that both procedures can access

- **Different Instructions ≈ Different Business Logic**: Each agent has its own instruction set, similar to each procedure having its own business rules

## What's Next

Steps 8 and 9 (optional) add a web-based UI and token server for production deployments. Step 10 (optional) introduces MCP (Model Context Protocol) for dynamic tool discovery.
