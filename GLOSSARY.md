# Matcha

A dating site where users build a profile, find each other, send crushes, and chat once the crush is mutual.

## Language

### People

**User**:
A person's account: identity, credentials and verification state.
_Avoid_: Account, member

**Verified User**:
A user who has confirmed their email address through the link sent at registration or a password reset; only verified users can log in.
_Avoid_: Active user, confirmed account

**Profile**:
The public part of a user: gender, sexual preference, biography, interest tags, photos, birth date and location.
_Avoid_: Card, page

**Complete Profile**:
A profile with every field filled, at least one photo, a profile picture and a location; only users with one browse, search, view others or appear to others.
_Avoid_: Active profile, finished profile

**Gender**:
Whether a user is a man or a woman.
_Avoid_: Sex

**Sexual Preference**:
The set of genders a user is interested in; when left unstated it is both.
_Avoid_: Orientation, looking for

**Compatible**:
Said of two users when each one's gender is in the other's sexual preference.
_Avoid_: Matching

**Birth Date**:
The day a user was born, never shown to others; their age is derived from it.
_Avoid_: Birthday, DOB

**Location**:
Where a user is considered to be, at neighbourhood precision, set from their device position with their consent or entered by hand.
_Avoid_: Position, address, coordinates

**Profile Picture**:
The one photo of a profile chosen to represent the user.
_Avoid_: Avatar, main photo

**Interest Tag**:
A short shared label for something a user likes, reusable across profiles.
_Avoid_: Hashtag, hobby, interest

**Common Tags**:
The interest tags two profiles share.
_Avoid_: Shared interests, overlap

**Suggestion**:
A compatible profile proposed to a user, ranked first by how close it is, then by common tags and popularity score.
_Avoid_: Match, recommendation

**Popularity Score**:
A public number from 0 to 100 measuring how much interest other users currently show in a user.
_Avoid_: Fame rating, rank, karma

### Relations

**Crush**:
One user's declared attraction to another user, which requires the sender to have a profile picture.
_Avoid_: Like, interest, wink

**Uncrush**:
Withdrawing a crush one has sent, which also silences every notification the other user would cause the withdrawer until the withdrawer sends a new crush.
_Avoid_: Unlike, dislike

**Connection**:
The state of two users who have each sent a crush to the other; only connected users can chat.
_Avoid_: Match, friendship

**Block**:
One user's decision to make another user and themself mutually invisible, which removes any crush between them.
_Avoid_: Ban, mute

**Fake Report**:
One user's claim that another user's profile is not a real person, which also blocks that user for the reporter.
_Avoid_: Flag, complaint

**Visit**:
One user viewing another user's profile.
_Avoid_: View, profile hit

### Activity

**Conversation**:
The one lasting chat thread between two users, open only while they are connected.
_Avoid_: Chat room, thread

**Message**:
A piece of plain text one user sends to another in their conversation.
_Avoid_: Chat, DM

**Notification**:
A real-time notice that someone crushed on, visited, messaged, connected with or uncrushed the user.
_Avoid_: Alert, event

**Online**:
Said of a user who currently has at least one live real-time channel open.
_Avoid_: Active, connected

**Last Seen**:
The moment a user stopped being online.
_Avoid_: Last connection, last login

**Meetup**:
A real-life meeting one connected user proposes to the other, with a time and a place, which the other accepts or declines and either can cancel.
_Avoid_: Date, event, appointment

### Relationships

- A **User** has exactly one **Profile**.
- A **Crush** goes from one **User** to another; two crushes in opposite directions form a **Connection**.
- An **Uncrush** of a connected user breaks the **Connection** and closes their **Conversation**, which reopens with its history if they connect again.
- A **Meetup** exists only between connected users and is cancelled when their **Connection** breaks.
- A **Block** or a **Fake Report** removes the crushes, and so the **Connection**, between the two users.
- Only **Compatible** users with a **Complete Profile** are suggested or found by search.
