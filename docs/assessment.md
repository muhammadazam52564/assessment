Dear Candidate,
I hope you’re doing well.
As the next step in our recruitment process, we would like you to complete the assessment test shared with you.
Kindly complete the assessment while screen recording your entire session, as the recording will be reviewed by our team as part of the evaluation process.
Please note that through this assessment, we would also like to understand your AI usage and how effectively you leverage AI-driven tools in your problem-solving and overall approach. The purpose is to assess your familiarity and comfort with AI as part of your working process.
Please ensure that the screen recording is clear and covers the complete duration of the assessment.
You can find the assessment details below in this email.

How we want you to work: record your AI process

\*We expect you to use AI tools (Claude or similar) for this assessment, and we want to see how you use them. How you think, give instructions, and handle the AI matters to us as much as the final result.

What we need:

Record your screen while you work, from your very first prompt to the end.
The recording must show the work as it happens: the instructions you give the AI, how you review its output, where you correct it or push back, and how you get around problems.
Do not build everything first and then record a walkthrough afterwards.
Do not send only your prompts or chat history. We want to watch the process live, not read a summary of it.
If you work across several sessions, record each one and send them in order.
Recordings do not need to be edited or polished. Raw is fine, and voice narration is optional but welcome.\*

(Background)
DataPhone is a VoIP provider in Brooklyn, NY. We sell hosted phone systems and SIP trunks, and every customer uses phone numbers (DIDs) that we buy from carriers.
Today our numbers live in three separate carrier portals, and nothing tells us in one place which customer owns which number. When a customer leaves, a number moves, or a port finishes, the record lives in someone's head or an email.
We buy numbers from three carriers, in order of volume:
Peerless Network (largest)
BulkVS
Bandwidth
Goal
Build a proof of concept (POC) that gives us one place to see, manage, buy and port in all our phone numbers across every carrier.
The POC is a success when our team can do the following without logging into any carrier portal:
Type any phone number and within seconds see which carrier it is with, which customer owns it, and everything that happened to it
Pull up a customer and see all of their numbers
Move a number from one customer to another, with a record of who did it and when
Search for and buy a new number
Start a port-in for a new customer and follow it until it completes
See a list of numbers we pay a carrier for but that no customer is using
Feature 1: All numbers in one place
Every number we hold at any carrier shows up in one list, and that list stays up to date on its own.
What I need:
One list of all numbers from all carriers
For each number: the carrier, the customer it belongs to (if any), its status, the date we got it, and the city/state it belongs to
Search by full or partial number, customer name, carrier, or status
Filter to show only unassigned numbers, only numbers of one customer, or only numbers at one carrier
Export the list to Excel/CSV
The list stay up to date
Feature 2: Assign numbers to customers
Every number belongs to one customer or to nobody, and our team can change that in a few clicks.
What I need:
Assign one number or many numbers at once to a customer
Move a number from one customer to another
Take a number back from a customer (for example, when they leave)
A customer page that lists all their numbers
For the POC, a simple customer list inside the app is fine. Later it will need to match our billing system (Rev.io)
Feature 3: Full history
Nothing that happens to a number is ever lost. We must be able to answer "who had this number, and when?" years from now.
What I need:
Every change is recorded: bought, ported in, assigned, moved, released, disconnected, note added, and changes the carrier made on its side
Each record shows what changed, from what to what, who did it (a person or the automatic carrier update), and the date and time
History can be viewed for one number or for one customer
History can never be edited or deleted, even by an admin
Feature 4: Buy new numbers
Our team can find and buy a number from inside the app, and it lands in the list already assigned to the right customer.
What I need:
Search available numbers by area code, city/state, or a pattern (for example, ending in 1000)
See results from our carriers side by side, with the price where the carrier gives it
Pick one or more numbers and buy them
Choose the customer at the time of purchase, so the number is assigned right away
Disconnect a number we no longer need, so we stop paying for it
Feature 5: Port in numbers for new customers
When a new customer brings their numbers from another provider, our team starts and tracks the whole port from inside the app.
What I need:
Check first whether the numbers can be ported
Create a port request with the customer's details: account name, service address, current provider, account number and PIN
Submit the port to the carrier we choose
A simple status everyone understands: Draft, Submitted, Date confirmed, Completed, Rejected, Cancelled
Show the confirmed port date (FOC) once the carrier gives it
If the carrier rejects the port, show the reason and let us fix and resubmit
A list of all open ports, so nothing gets forgotten
When a port completes, the numbers show up in the list already assigned to that customer
Must work with many carriers
The app must work the same way for our team no matter which carrier a number is with. Adding a fourth carrier later should not mean rebuilding the app.
The POC must connect to Peerless, BulkVS and Bandwidth
Our team should not need to know how each carrier works; the screens are the same for all
Each carrier's login details are kept safe and are not visible to regular users
Not part of the POC
Porting numbers out to other providers
E911 address management
Billing, invoicing or syncing with Rev.io
Changing call routing in our phone system (FusionPBX)
SMS/messaging setup
A customer-facing portal; the POC is for DataPhone staff only
What we want back from you
We are leaving the technical design to you on purpose; how you approach it matters as much as the result.
Your proposal should cover:
How you would build it, and why you chose that approach
How the app will handle three carriers that work very differently, and how a new carrier would be added later
How you will keep history complete and trustworthy
How you will keep our list in sync with the carriers
What you would build first, and a timeline in phases
The risks you see and what you would do about them
The questions you need answered before you start
At the end of the POC we expect:
A live demo of each feature above, using our real carrier accounts
The source code in our GitHub
Short notes on how to run it and how to add a new carrier
Reference material
Peerless Network API: WSDL files (APIService.wsdl and APIServicePortType.wsdl), sent separately
BulkVS API documentation
Bandwidth: searching and ordering numbers
Test accounts and API credentials for each carrier will be provided once the proposal is approved.

Note: Please submit the assessment within 3 days of receiving this email, as due to the high volume of applications, your application will be automatically dropped from our system after the deadline.

Best of luck!
