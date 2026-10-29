Hi Arlie,

The Deal Stacker automation (AUC-102, AUC-104 and AUC-109) is ready for UAT. When an opportunity moves to
UNDER ANALYSIS in GHL, it now builds the property's Google Drive deal folder automatically. Could you
run through the steps below and tell me what passes or fails?

**Before you start**

- During UAT, the GHL workflow only reacts to opportunities whose **name contains "TEST"** (for
  example "TEST 12 High Street"). Other cards are ignored for now.
- Use an opportunity that **does not already have a folder** in Property Deals. The automation only
  builds the folder, the files and the photos the first time.
- On the opportunity, fill in:
  - **Property Engine Link**: a property that is currently listed on PropertyEngine
  - **Rightmove Link**: optional
  - **Postcode**
  - **Guide Price**
  - **New Market Value**
  - **Number of Bed (Potential)**
  - **LHA Rate (weekly)**

**Links you will need**

- Property Deals folder (where the deal folder appears):
  https://drive.google.com/drive/folders/1FXPz2sCEBJdcVerC6RK7an4yKO5ThneK
- Master Deal Stacker (template, for comparison):
  https://docs.google.com/spreadsheets/d/1Pdow8Q2bjEtN4lNbq-KzBAyDnjoZOn8u9niE_T9Q0aY
- Sample Schedules (template, for comparison):
  https://docs.google.com/spreadsheets/d/1gUafgwbDwn174LwVqeP-JqrXLfbvS4ICt9uNmgAlmpQ
- The property itself: the Property Engine Link on the card

**Steps**

1. In GHL, open the pipeline **BMV Deals Follow up** and drag your TEST opportunity into
   **UNDER ANALYSIS**. Note the time.
2. Open the **Property Deals** folder (link above) and refresh it.

**What to check, and how long to wait**

| Wait | Where | What you should see |
|---|---|---|
| About 10 seconds | Property Deals | A new folder named exactly like the opportunity, containing four subfolders: **Photos, Videos, Floorplan, Comparables** (AUC-102) |
| About 10 seconds | The GHL opportunity (reopen it) | **Google Drive Link** is filled in and opens that folder (AUC-102) |
| About 20 seconds | Deal folder | **Stacker - [opportunity name]**, a copy of the Master (AUC-109 scenario 1) |
| About 20 seconds | Stacker, tab **Numbers Overview All Options** | **C10** = Guide Price, **C44** = New Market Value, **F69** = Number of Bed (Potential), **F72** = LHA Rate (weekly), all matching the card (AUC-109 scenario 2) |
| About 20 seconds | Deal folder | **SoW - [opportunity name]**, a copy of Sample Schedules (AUC-109 scenario 3) |
| — | Videos | Empty. It is filled manually later (AUC-104 scenario 3) |
| 3 to 5 minutes | Photos | The listing's photos, named `01-exterior.jpg`, `02-kitchen.jpg` and so on, the same pictures as on the PropertyEngine page (AUC-104 scenario 2) |
| 3 to 5 minutes | Floorplan | `floorplan-1.png`, the floor plan from PropertyEngine, if the listing has one (AUC-104 scenario 4) |
| 3 to 5 minutes | Comparables | `comparables.csv` from PropertyData: the same property type, within 0.5 miles, sold in the last 12 months, freehold only (AUC-104 scenario 5) |
| 3 to 5 minutes | Deal folder | `PropertyEngine summary.txt`: address, asking price, beds and baths, EPC and agent, matching the PropertyEngine page |

The photos take the longest because each one is downloaded and uploaded separately. If nothing
appears in Photos after **10 minutes**, please send me the opportunity name and the time you moved it.

**Extra checks**

1. **Move the same card again.** Move it out of UNDER ANALYSIS and back in. There should be **no
   second folder**, and no second Stacker or SoW. Only the Drive link on the card is refreshed.
2. **No Property Engine Link.** Use a new TEST opportunity with only a Rightmove Link. The photos
   should still arrive.
3. **Empty field.** Use a new TEST opportunity with no Guide Price. C10 in the Stacker should keep
   the template's value instead of being blanked.

**Known limitations**

- The photos are PropertyEngine's standard size, not full resolution.
- If PropertyEngine cannot find the property (no link and the address is not on the market), the
  folder and Stacker are still created but Photos, Floorplan and Comparables stay empty.

Please reply with pass or fail for each row, plus screenshots of anything that looks wrong.

Thanks,
Julian
