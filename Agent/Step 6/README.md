# Step 6: Booking Functionality

## What Changed in This Step

This step expands the agent's capabilities by adding booking management functionality. Users can now schedule service appointments, check available dates, and view existing bookings - all through natural conversation.

### New/Modified Files

- **`requirements.txt`**: No changes (uses same dependencies as Step 5)

- **OpenEdge Side**: New booking service components:
  - **`bookingHandler.cls`**: ABL web handler for booking operations (GET/POST)
  - **`bookingService`**: WEB transport service with URIs:
    - `/booking` - Get booking by registration
    - `/booking/{method}` - Get next available slot or save booking

- **`OEDatabaseDriver.py`**: Enhanced with booking operations:
  - **`Booking` dataclass**: Structured type for booking records
  - **`get_next_available_booking()`**: Returns next available appointment date
  - **`save_booking()`**: Creates a new booking
  - **`get_booking()`**: Retrieves existing booking for a car

- **`agent.py`**: Expanded with booking tools:
  - **`get_the_date_today()`**: Returns current date in conversational format
  - **`get_next_available_booking_date()`**: Finds next available slot
  - **`book_appointment()`**: Creates a booking
  - **`get_booking()`**: Retrieves existing booking
  - **`date_to_long_string()`**: Helper to format dates nicely ("4th November 2025")

### Code Overview

#### New Booking Dataclass (OEDatabaseDriver.py)

```python
@dataclass
class Booking:
    def __init__(self, booking_date: Optional[date] = None, description: str = ""):
        self.booking_date = booking_date
        self.description = description
    booking_date: date
    description: str
```

#### New Driver Methods (OEDatabaseDriver.py)

```python
def get_next_available_booking(self, earliest_date: date) -> date:
    """Query OpenEdge for next available booking slot"""
    url = f"{BASE_URL}booking/next"
    params = {"earliestDate": earliest_date.isoformat()}
    response = requests.get(url, params=params)
    return datetime.fromisoformat(response.json()["date"]).date()

def save_booking(self, reg: str, date: date, description: str) -> bool:
    """Create a new booking"""
    url = f"{BASE_URL}booking"
    payload = {"reg": reg, "date": date.isoformat(), "description": description}
    response = requests.post(url, json=payload)
    return response.status_code == 200

def get_booking(self, reg: str) -> Optional[Booking]:
    """Get existing booking for a registration"""
    url = f"{BASE_URL}booking"
    response = requests.get(url, params={"reg": reg})
    if response.status_code == 200:
        data = response.json()
        return Booking(
            booking_date=datetime.fromisoformat(data["bookingDate"]).date(),
            description=data.get("description", "")
        )
    return None
```

#### New Agent Tools (agent.py)

```python
@function_tool
async def get_the_date_today(self):
    date_str = self.date_to_long_string(date.today())
    return f"The date today is {date_str}"

@function_tool
async def get_next_available_booking_date(
    self, 
    earliest_date: Annotated[date, "Earliest date for booking"]
):
    next_date = driver.get_next_available_booking(earliest_date)
    return f"The next available booking date is {self.date_to_long_string(next_date)}"

@function_tool 
async def book_appointment(
    self, 
    reg: Annotated[str, "Car registration number"], 
    date: Annotated[date, "Date for the appointment"], 
    description: Annotated[str, "Description of the appointment"]
):
    if driver.save_booking(reg.upper().replace(" ", ""), date, description):
        return f"Appointment booked for {self.date_to_long_string(date)} with description: {description}"
    return "Failed to book appointment, please try again later"

@function_tool
async def get_booking(self, reg: Annotated[str, "Car registration number"]):
    booking = driver.get_booking(reg.upper().replace(" ", ""))
    if booking is None:
        return "No appointment found"
    return f"Next appointment is on {self.date_to_long_string(booking.booking_date)} with description: {booking.description}"
```

## What This Implements

**Complete Booking Management System**: This step adds end-to-end appointment scheduling:

- **Date/Time Handling**: Works with Python `date` objects and converts to/from ISO format for API calls
- **Availability Checking**: Queries the backend for the next available appointment slot
- **Booking Creation**: Saves appointments with registration, date, and description
- **Booking Retrieval**: Looks up existing bookings by registration
- **Natural Date Formatting**: Converts dates to conversational format ("4th November 2025" instead of "2025-11-04")

The agent can now handle conversations like:
- "When is my next appointment?" → Calls `get_booking()`
- "When's the next available slot?" → Calls `get_next_available_booking_date()`
- "Book me in for an oil change next Tuesday" → Calls `book_appointment()`

## How This Helps Build Towards the Final Solution

This step significantly expands the agent's usefulness:

1. **Business Logic Integration**: Demonstrates how to connect the agent to real business workflows (appointment scheduling), not just data lookup

2. **Type Conversion**: Shows how to handle complex types:
   - Python `date` objects in function parameters
   - Conversion to ISO strings for API calls
   - Parsing ISO strings back to `date` objects
   - Formatting for human-readable output

3. **Multi-Step Workflows**: Enables complex interactions:
   ```
   User: "I need to book a service"
   Agent: [Calls get_next_available_booking_date()]
         "Next available is 4th November"
   User: "Perfect, book it for an oil change"
   Agent: [Calls book_appointment()]
         "Booked your oil change for 4th November"
   ```

4. **OpenEdge Backend Expansion**: Shows the pattern for adding new business entities:
   - Create ABL handler (bookingHandler)
   - Create WEB service (bookingService)
   - Add Python driver methods
   - Expose as agent tools
   - No changes to agent infrastructure!

5. **Parameter Annotations**: The `Annotated[date, "Description"]` pattern helps the LLM understand:
   - What type each parameter should be
   - What the parameter means
   - How to extract it from user speech

## Testing This Step

### Prerequisites
1. Deploy the new OpenEdge components:
   - `bookingHandler.cls`
   - `bookingService` (with URIs configured)

2. Restart PASOE to load the new service

### Test Scenarios

Start the agent:
```powershell
py main.py dev
```

Connect via [LiveKit Agents Playground](https://agents-playground.livekit.io/)

**Test 1: Check Today's Date**
- You: "What's today's date?"
- Agent: "The date today is [formatted date]"

**Test 2: Find Next Available Slot**
- You: "When's the next available appointment?"
- Agent: [Calls function] "The next available booking date is [date]"

**Test 3: Create Booking**
- You: "Book me in for an annual service next week"
- Agent: [Gets next date, then books] "Appointment booked for [date] with description: annual service"

**Test 4: Check Existing Booking**
- You: "Do I have any appointments?"
- Agent: [Looks up] "Your next appointment is on [date] for [description]"

### Verification
Use `OpenEdge\ViewBookings.w` (if available) or query the database directly to confirm:
- Bookings are being created with correct dates
- Descriptions are stored properly
- Registration numbers match correctly

## Key Concepts for OpenEdge Developers

- **Booking Dataclass ≈ Booking TEMP-TABLE**: Similar to:
  ```abl
  DEFINE TEMP-TABLE ttBooking
    FIELD bookingDate AS DATE
    FIELD description AS CHARACTER.
  ```

- **Date Handling**: Python's `date` type is similar to ABL's DATE type, but requires explicit parsing:
  - ABL: `dDate = DATE("11/04/2025")`
  - Python: `date_obj = datetime.fromisoformat("2025-11-04").date()`

- **ISO Format**: The ISO date format (YYYY-MM-DD) is language-agnostic and avoids confusion between US (MM/DD/YYYY) and UK (DD/MM/YYYY) formats

- **Multiple Tools**: Just like having multiple procedures in a `.p` file, the agent can have multiple function tools - the LLM decides which to call

## What's Next

Step 7 introduces multi-agent architecture, splitting responsibilities between an Account Agent (handles car lookup/creation) and a Booking Agent (handles appointments), with automatic handoff between them.
