# A Service Oriented ViewModel

## Jacob Foshee
## MonkeySpace 2013


# Summary
- Service not reinvented by 3rd party
- VM not reinvented by 2nd device
- VM Responsible for wrapping DTOs 
  *and* User's Intent
- Patterns yield one-liners for mapping VM property(s) to DTO(s)
- Email me if interested in the code: 
  ## jacobf@gmail.com


# Goals of the Talk
- Give back
- Synergy of ideas
- Result of a natural refactoring
- API Review
- On-Demand (Lazy) Open-Source

----

# Goals of SOA+MVVM
- Take advantage of strong types and reflection
- Non-UI-blocking 
- Use standard MVVM patterns (`INotifyPropertyChanged`)
- Observe little difference between a property being
  updated from a local calculation vs. fetching from a service. 


# Common Issues
- Synchronize local VM properties with remote persistence
- Disable part of UI while waiting for response 
  (e.g. avoid double submit)
- Errors (log or display, don't allow an exception 
  to go uncaught *especially on background thread*)
	- Developer needs details
	- User usually just needs to check connection

---

# Background
- ## MVVM
- ## ServiceStack


# MVVM

	public class PropertyChangedEventArgs : EventArgs
	{
	    public virtual string PropertyName { get; }
	}
	
	public interface INotifyPropertyChanged
	{
	    event PropertyChangedEventHandler PropertyChanged;
	}


# ServiceStack

- Message Based
- `IReturn <T>`
- `IHasLongId`
- `ServiceClientBase`
- `ILog`

---

# Patterns


# Property is DTO
- ### Request & Set Property
- ### Post & Set Property


# Property is part of DTO
- ### Post Update for Property 
- ### (and disable while waiting)


# Property is `IList` of DTO
- ### Post & Add
- ### Delete & Remove


# Plus 
- ### Raise PropertyChanged 
- ### Log Errors 
## For all of the above

---

# A DTO

    public class Monkey : IHasLongId, 
                          IReturn<Monkey>
    {
        public long   Id        { get; set; }
        public bool   IsPrimate { get; set; }
        public string Text      { get; set; }
    }


# Initialize Property

    public class MonkeyRequest : 
      IReturn<Monkey> { … }

    public class MonkeyViewModel : 
      ServiceClientViewModelBase
    {
      public Monkey Monkey { get; set; }
        
      public void OnInitialize(long id) {
        var req = new MonkeyRequest(id);
        RequestAndSetProperty(req, () => Monkey);
      }
    }


# New Record with …

    Monkey Monkey        { get; set; }
    string Text          { get; set; }
    bool   IsTextEnabled { get; set; }

    void SaveText() {
        if (Monkey.Id == 0) {
            Monkey.Text = Text;
            PostAndSetProperty(Monkey, () => Monkey,
              "IsTextEnabled");
        }
        else // post update …
    }


# Update Part of a Record

    public class MonkeyTextUpdate : IReturn<Monkey> {
      public long   Id   { get; set; } 
      public string Text { get; set; } 
    }
    
    // …
    else /* Monkey.Id != 0 */ {
      var req = new MonkeyTextUpdate(Monkey.Id, Text);
      PostAndSetProperty(req, () => Monkey, 
        "IsTextEnabled");
    }


# List of DTOs

    public class MonkeyListRequest : 
      IReturn<IList<Monkey>> { ... }

    public class MonkeyListViewModel : ServiceClientViewModelBase
    {
        public IList<Monkey> Monkeys { get; set; }
        
        void RequestMonkeyList() {
          var req = new MonkeyListRequest();
            RequestAndSetProperty(req, () => Monkeys);
        }
    }


# Add / Remove

    IList<Monkey> Monkeys { get; set; }

    public void AddMonkey() {
        var newMonkey = new Monkey { … };
        PostAndAdd(newMonkey, () => Monkeys);
    }

    public void RemoveMonkey(Monkey monkey) {
        DeleteAndRemove(monkey, () => Monkeys);
    }


# Add Advanced    

    IList<Monkey> Monkeys { get; set; }
    string NewMonkeyText  { get { … }; set { … Notify(() => NewMonkeyText); }
    bool CanAddMonkey     { get { … }; set { … Notify(() => CanAddMonkey); }

    public void AddMonkey() {
        CanAddMonkey = false;
        var newMonkey = new Monkey { Text = NewMonkeyText };
        PostAndAdd(newMonkey, () => Monkeys, 
        () => { // On Success
            NewMonkeyText = null;
            CanAddMonkey = true;
        }, // On Error
        () => CanAddMonkey = true);
    }


## Update Entire Record w/ wrapped DTO property

    Monkey Monkey  { get; set; }
    bool IsPrimate {
        get { return Monkey.IsPrimate; }
        set { Monkey.IsPrimate = value; }
    }
    bool IsPrimateIsEnabled { get; set; }

    public void ToggleIsPrimate() {
        bool newPrimate = !IsPrimate;
        // If the update fails, we haven't changed 
        // the local copy misleadingly
        var req = new Monkey(Monkey) { 
          IsPrimate = newPrimate };
        PostUpdateForProperty(req, () => IsPrimate, 
          newPrimate);
    }

---

# Considerations

## When & Where…


# Service
- Integrity of the Persistent System
- Up to date
- Not reinvented by 3rd Party
- Example: Sign-off operations should be bundled together as a single service request


# ViewModel
- Integrity of the Presentation
- **User's Intent**
- Out of date (snapshot)
	- Only update part of record
	- Don't send update if none intended (nothing changed)
- Not reinvented for another device


# Examples
- One user on multiple devices
- Toggle checkbox
	- The toggling should be done in the VM so two "toggles" don't cancel out accidentally
	- Can represent failed intent by not checking the box
- Text box
	- Shouldn't erase the text when save failed, so need to represent failed intent another way


# More VM Thoughts
- Implicit Save
- Fear the prospect of debugging network-related race conditions
- Do you wrap DTO properties?


# View [Controller]
- Respect the ViewModel
- Respect the Device/Paradigm
- Widget-specific mechanics
- Update on Main/UI Thread

---

# Client Platform

- Preserve Attribute on DTO & VM assemblies

      #if MONOTOUCH
      [assembly: MonoTouch.Foundation.Preserve(AllMembers = true)]
      #endif

- Update on Main/UI Thread

		static void HandlePropertyChangedFromEventThread(
			this ICanHandlePropertyChanges source, 
			object sender, PropertyChangedEventArgs e)
		{
			source.InvokeOnMainThread(() => {
			  if (source.IsViewLoaded) 
				source.HandlePropertyChanged(e.PropertyName);
			});
		}
	

---

# Future Work
- Email me for the code: 
  ##<jacobf@gmail.com>
- ViewModel Factory stuff (hint: Segues)
- Convention for SaveFailedFor____ Properties
- Pattern for wrapping collection of DTOs
- Pub/Sub patterns


# Acknowledgments
- MonkeySpace Organizers & Sponsors
- New York Presbyterian Hospital (*now hiring…*)

<br />

### Presented w/ **reveal.js**
