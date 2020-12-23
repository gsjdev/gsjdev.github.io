## 0. Prepare project file
- Place/Create project [API DevKit > developing > ]
- Set [Debug > Command] : ARCHICAD 23.exe 
- Set [Debug > Argument]: -DEMO
- Set [Linker > Output path]: ARCHICAD 23\Add-Ons



## 1. Create a class for your palette 

### Inherit DG class
- Inherit DG::Palette class
- Inherit DG::PaletteObserver class
``` c++
#include "DG.h"
#include "DGModule.hpp"

class SamplePalette:
	public DG::Palette,
	public DG::PanelObserver
{
public:
	SamplePalette();
	~SamplePalette();
};

```

### Definition of Constructor & Destructor
- Define constructor: 
	- Attach to panel observer
- Define destructor: 
	- Detach to panel observer

``` c++
SamplePalette::SamplePalette():
DG::Palette(ACAPI_GetOwnResModule(), 32500, ACAPI_GetOwnResModule())
{
	Attach (*this);
	BeginEventProcessing ();
}


SamplePalette::~SamplePalette()
{
	Detach(*this);
	EndEventProcessing ();	
}
```


### GRC: Definition of Palette Resources
- Define palette status:
	- flag
	- coordinate and size

```
'GDLG'  resID  Palette [| frameFlag | growFlag | captionFlag | closeFlag]  x  y  dx  dy  "dlgTitle" 
{
      dialogItem1
          ...
      dialogItemi
          ...
      dialogItemn
}

x and y are the pixel coordinates of the upper left corner of the dialog

```

``` c++
'GDLG'  32500  Palette | close      0    0  400  300  "" 
{
}
```

- Define Menu Strings
	- Option > SamplePalette > Open Palette

``` c++
'STR#' ID_MENU_STRINGS "Menu strings" {
/* [  1] */		"SamplePalette"
/* [  2] */		"Open Palette"
}
```

--- 


## 2. Create palette instance

<center>

![](pic/palette1.png)

</center>


### Instance member
- Static member for instance
	- Use GS::Ref<> type or  * pointer to be single instance & use memory dynamically
	- Singleton: https://en.wikipedia.org/wiki/Singleton_pattern
- Create static functions to handle static member


``` c++
class SamplePalette:
	public DG::Palette,
	public DG::PanelObserver
{
public:
	SamplePalette();
	~SamplePalette();
	
	static SamplePalette& GetInstance();
	static bool HasInstance();
	static void CreateInstance ();
	void ShowPalette ();
	
private:	
	static GS::Ref<SamplePalette> instance;
    // static SamplePalette* instance;
};
```

### Definition of member & function
- Declare static member
- Define functions
	- Use ACAPI_KeepInMemory(true) when there are any global or static variables.

``` c++
GS::Ref<SamplePalette> SamplePalette::instance;
//SamplePalette* SamplePalette::instance = nullptr;

SamplePalette& SamplePalette::GetInstance()
{
	return *instance;
}

bool SamplePalette::HasInstance()
{
	return instance != nullptr;
}

void SamplePalette::CreateInstance()
{
	instance = new SamplePalette;
	ACAPI_KeepInMemory (true);
}

void SamplePalette::ShowPalette ()
{
	DG::Palette::Show();
	DG::Palette::BringToFront ();
}

```


### Menu Command Handler
- Check whether the palette instance already exists or not
- Create instance and show palette

``` c++
GSErrCode __ACENV_CALL MenuCommandHandler (const API_MenuParams *menuParams)
{
    switch (menuParams->menuItemRef.menuResID)
    {
        case 32500:
        {
            switch (menuParams->menuItemRef.itemIndex){
                case 1:
                {
                    if(!SamplePalette::HasInstance ()){
                        SamplePalette::CreateInstance ();
                    }
                    
                    SamplePalette::GetInstance ().ShowPalette();
                }
                break;
            }
        }
    }
}
```

--- 


## 3. Create a palette item : SingleSelList

<center>

![](pic/palette2.png)

</center>


### GRC: Definition of item resouce
- Define single selection listbox item

``` c++
SingleSelList x  y  dx  dy  fontSpec  partItems  [scrollType]  itemSize  [headerFlag    headerSize]
```

``` c++
'GDLG'  32500  Palette | close      0    0  400  300  "" {
/* [  1] */ SingleSelList	 2   2  396  296  SmallPlain  PartialItems  21  HasHeader  21
}
```

### Definition of ListBox item
- Inherit DG::ListBoxObserver
	- for event callback
- Add listbox type member and its unique index

``` c++
class SamplePalette:
	public DG::Palette,
	public DG::PanelObserver,
	public DG::ListBoxObserver
{
private:	
	enum
	{
		listBoxID = 1,
	};
	
	DG::SingleSelListBox listBox;

```

### Initialization of Listbox item
- Initialize the listbox in the constructor
- Attach to listbox observer

``` c++
SamplePalette::SamplePalette():
DG::Palette(ACAPI_GetOwnResModule(), 32500, ACAPI_GetOwnResModule()),
listBox(GetReference(), listBoxID)
{
	Attach (*this);
	BeginEventProcessing ();
	listBox.Attach(*this);
}

```

---


## 4. Add element info. to listbox

<center>

![](pic/palette3.png)

</center>

### Override Observers Function
- Go to DG::PanelObserver definition
	- Copy and Paste PanelOpened() func.

``` c++
class SamplePalette:
	public DG::Palette,
	public DG::PanelObserver,
	public DG::ListBoxObserver
{
public:
    virtual void PanelOpened (const DG::PanelOpenEvent& ev) override;
};
```

### Get Opened Event and Aet Element info.
- Get all element on the project
- Append item to listbox and set the item text

``` c++
void SamplePalette::PanelOpened (const DG::PanelOpenEvent& ev)
{
	UNUSED_VARIABLE(ev);

	GSErrCode err = NoError;
	GS::Array<API_Guid> elemGuids;
	
	err = ACAPI_Element_GetElemList(API_ZombieElemID, &elemGuids);
	if(err != NoError)
		return;
	
	for(API_Guid guid: elemGuids)
	{
		API_Element elem = {};
		elem.header.guid = guid;
		
		err = ACAPI_Element_Get(&elem);
		if(err != NoError)
			continue;
		
		listBox.AppendItem();
		GS::UniString text = ElemID_To_Name(elem.header.typeID);
		listBox.SetTabItemText(DG::SingleSelListBox::BottomItem, 1, text);
	}
}

```

---

## 5. Add userdata to listbox items

### Allocate new memory for userdata
- Set userdata to listbox item using SetItemValue(()
- Delete all userdata to release its memory when the instance is destructed 


``` c++
SamplePalette::~SamplePalette()
{
	Detach(*this);
	EndEventProcessing ();
	
	for(int i = 1 ; i <= listBox.GetItemCount(); i++)
	{
		API_Guid* pGuid = (API_Guid*)listBox.GetItemValue(i);
		if(pGuid != nullptr){
			delete pGuid;
		}
	}
	
	listBox.DeleteItem(DG::SingleSelListBox::AllItems);	
	listBox.Detach(*this);
}

// event callback 
void SamplePalette::PanelOpened (const DG::PanelOpenEvent& ev)
{
	UNUSED_VARIABLE(ev);

	GSErrCode err = NoError;
	GS::Array<API_Guid> elemGuids;
	
	err = ACAPI_Element_GetElemList(API_ZombieElemID, &elemGuids);
	if(err != NoError)
		return;
	
	for(API_Guid guid: elemGuids)
	{
		API_Element elem = {};
		elem.header.guid = guid;
		
		err = ACAPI_Element_Get(&elem);
		if(err != NoError)
			continue;
		
		listBox.AppendItem();
		GS::UniString text = ElemID_To_Name(elem.header.typeID);
		listBox.SetTabItemText(DG::SingleSelListBox::BottomItem, 1, text);

		API_Guid* pGuid = new API_Guid(guid);
		listBox.SetItemValue(DG::SingleSelListBox::BottomItem, (DGUserData)pGuid);		
	}
	
}
```

---


## 6. Get event when a listbox item is clicked & Get userdata 

<center>

![](pic/palette4.png)

</center>

### Override ListBoxObserver func
- Go to ListBoxObserver definition
	- Override ListBoxClicked()

``` c++
class SamplePalette:
	public DG::Palette,
	public DG::PanelObserver,
	public DG::ListBoxObserver
{
public:
	virtual void ListBoxClicked (const DG::ListBoxClickEvent& ev) override;
};
```


### Get event & get userdata 
- Get userdata(guid) and camera jump to the element assigned the guid and select it.

``` c++
// Event Callback
void SamplePalette::ListBoxClicked (const DG::ListBoxClickEvent& ev)
{
	if(ev.GetSource()==&listBox)
	{		
		short selecedItem = listBox.GetSelectedItem();
		
		API_Guid* pGuid = (API_Guid*)listBox.GetItemValue(selecedItem);
		if(pGuid == nullptr)
			return;

		API_Neig neig = {};
		neig.guid = *pGuid;		
		ACAPI_Element_DeselectAll();
		ACAPI_Element_Select({neig}, true);
	
		GS::Array<API_Guid> guidArray;
		guidArray.Push(*pGuid);
		ACAPI_Automate (APIDo_ZoomToElementsID, &guidArray);
	}	
}
```

---


## 7. Delete the instance when the palette closed

### Override PanelObserver function
- Go to PanelObserver definition
	Copy & Paste PanelCloseRequested()

``` c++
class SamplePalette:
	public DG::Palette,
	public DG::PanelObserver,
	public DG::ListBoxObserver
{
public:
	void ClosePalette();
	virtual	void PanelCloseRequested (const DG::PanelCloseRequestEvent& ev, bool* accepted) override;
};
```

### Definition of override function

``` c++
void SamplePalette::ClosePalette()
{
	instance = nullptr;
    // delete instance;
}


// Event Callback
void SamplePalette::PanelCloseRequested (const DG::PanelCloseRequestEvent& ev, bool* accepted)
{
	UNUSED_VARIABLE(ev);
	ClosePalette();	
	*accepted = true;
}


```
