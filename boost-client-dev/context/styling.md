# Styling Guide (Tachyons + Boost Design System)

## Golden rules

1. **NEVER use Tailwind classes** — this project uses Tachyons, not Tailwind
2. **NEVER use inline styles** for properties that have utility classes
3. **NEVER invent class names** — only use classes that exist in Tachyons or `styles.global.less`
4. **NEVER recreate existing components** — search `src/components/` first

---

## Spacing (Tachyons standard scale: 0=0, 1=0.25rem, 2=0.5rem, 3=1rem, 4=2rem, 5=4rem, 6=8rem, 7=16rem)

```
ma{0-7}  pa{0-7}   — all sides
mt mb ml mr        — margin per side
pt pb pl pr        — padding per side
pv ph mv mh        — vertical/horizontal shorthand
```

## Flexbox

```
flex  inline-flex  flex-column  flex-row  flex-wrap
items-center  items-start  items-end  items-baseline
justify-center  justify-between  justify-around  justify-start  justify-end
self-center  self-start  self-end
flex-auto  fg1  fg2
```

## Custom gap (boost extension)

```
flex-gap-1  (0.25rem)
flex-gap-2  (0.5rem)
flex-gap-3  (1rem)
flex-gap-4  (2rem)
```

## Font sizes (custom extension: exact px)

```
fs8  fs9  fs10  fs11  fs12  fs13  fs14  fs15  fs16
fs17  fs18  fs19  fs20  fs22  fs24  fs30  fs40  fs54
```

## Typography

```
b  fw1-fw9  i  ttu  ttl  ttc  ttn
truncate  tc  tl  tr  lh-copy  lh-title  lh-solid
```

---

## Color system

**Only use boost-* color classes. Never hardcode hex values as class names.**

Pattern: `boost-{color}` (text) | `bg-boost-{color}` (bg) | `b--boost-{color}` (border)

### Primary (orange)
```
boost-primary          bg-boost-primary          b--boost-primary
boost-primary-light    boost-primary-dark
boost-primary-30       boost-primary-15
```

### Secondary (grays)
```
boost-secondary         bg-boost-secondary
boost-secondary-light   boost-secondary-dark
boost-secondary-70  boost-secondary-50  boost-secondary-30
boost-secondary-20  boost-secondary-15  boost-secondary-10  boost-secondary-05
```

### Status
```
boost-success  boost-success-dark  boost-success-light
boost-danger   boost-danger-dark   boost-danger-light
boost-warning  boost-warning-dark  boost-warning-light
boost-blue     boost-blue-dark     boost-blue-light
```

### Subtle backgrounds (for cards/sections)
```
bg-boost-primary-subtle   bg-boost-secondary-subtle
bg-boost-success-subtle   bg-boost-danger-subtle
bg-boost-warning-subtle   bg-boost-blue-subtle
```

---

## Icon system

**Only use icon names from `src/icons.global.less`.**

```tsx
import LMIcon from 'components/lawmatics/lm_icon';

// Preferred
<LMIcon icon="edit" className="fs14 boost-secondary-70" />
<LMIcon icon="edit" action={() => handleEdit()} />

// Direct (for simple usage)
<i className="icon-trash fs14 boost-danger" />
```

### Valid icon names (partial list)
```
edit  trash  plus  minus  clear  checkmark  search  filter
arrow-down  arrow-up  arrow-left  arrow-right
chevron-left  chevron-right  chevron-down  chevron-up
settings  calendar  email  sms  automation  tasks
duplicate  archive  preview  hidden  sort  drag-handle
save  share  link  redo  undo  warning  warning-circle
bell  note  merge  send  ai-write  zoom
```

---

## Component library (MUST USE before creating new components)

### Buttons
```tsx
import BoostButton from 'components/boost/boost_button';

<BoostButton
  title="Save"
  theme="success"    // default | success | danger | warning | gray | blue | hollow | basic
  size="default"     // default | small | smaller | tiny | large | hyper | wide | iconOnly
  icon="save"        // optional
  action={() => {}}  // omit for type="submit"
  disabled={false}
  loading={false}
/>
```

### Key components — always use these instead of custom alternatives

| Need | Component | Import path |
|------|-----------|-------------|
| Button | `BoostButton` | `components/boost/boost_button` |
| Icon | `LMIcon` | `components/lawmatics/lm_icon` |
| Loading | `LMLoader` | `components/lawmatics/lm_loader` |
| Text input | `BoostInput` | `components/boost/boost_input` |
| Select | `LMSelect` | `components/boost/lm_select` |
| Checkbox | `LMCheckbox` | `components/lawmatics/lm_checkbox` |
| Toggle | `LMToggleCheckbox` | `components/lawmatics/lm_toggle_checkbox` |
| Radio | `LMRadioButton` | `components/lawmatics/lm_radio_button` |
| Textarea | `LMTextarea` | `components/lawmatics/lm_textarea` |
| Date picker | `LMDatePicker` | `components/lawmatics/lm_date_picker` |
| Time picker | `LMTimePicker` | `components/lawmatics/lm_time_picker` |
| Popover | `LMPopover` | `components/lawmatics/lm_popover` |
| Dropdown | `LMDropdown` | `components/lawmatics/lm_dropdown` |
| Card | `LMCard` | `components/lawmatics/lm_card` |
| Tabs | `LMTabs` | `components/lawmatics/lm_tabs` |
| Tooltip | `LMHelptip` | `components/lawmatics/lm_helptip` |
| Delete dialog | `LMDeleteDialog` | `components/lawmatics/lm_delete_dialog` |
| Table | `BoostTable` | `components/boost/boost_table` |
| Info bar | `LMInfoBar` | `components/lawmatics/lm_info_bar` |

---

## LMLayers (slide-out panels) — strict structure required

```tsx
import { useModalContext } from 'components/boost/boost_modal';
import LMLayersHeader from 'components/lawmatics/lm_layers/lm_layers_header';
import LMLayersContent from 'components/lawmatics/lm_layers/lm_layers_content';
import LMLayersFooter from 'components/lawmatics/lm_layers/lm_layers_footer';

// Wrapper MUST have: flex flex-column flex-auto
<form onSubmit={handleSubmit} className="flex flex-column flex-auto">
  <LMLayersHeader title="Add Invoice" />
  <LMLayersContent>
    {/* fields — NO vertical margins on direct children, gap is built-in */}
  </LMLayersContent>
  <LMLayersFooter>
    <BoostButton title="Save" theme="success" />
  </LMLayersFooter>
</form>
```

---

## Common patterns

```tsx
// Card container
<div className="bg-boost-white br6px nice-shadow pa3">...</div>

// Section header
<div className="fs10 ttu b boost-secondary-70 mb2">Section Title</div>

// Clickable row
<div className="flex items-center pa2 pointer dim br3 hover-bg-boost-secondary-05">
  <LMIcon icon="edit" className="fs14 mr2" />
  <span className="fs14">Edit</span>
</div>

// Divider
<div className="bb b--boost-secondary-10 w-100" />

// Loading state
<LMLoader />          // full
<LMLoader small />    // small
<LMLoader tiny />     // tiny bars
```

---

## Common AI mistakes — avoid these

```tsx
// WRONG: Tailwind classes (don't exist here)
className="text-gray-500 gap-4 rounded-lg p-4"

// WRONG: Invented color names
className="text-primary bg-light-gray"

// WRONG: Inline styles for common properties
style={{ display: 'flex', marginTop: '16px' }}

// WRONG: Made-up icon names
<i className="icon-pencil" />  // use icon-edit
<i className="icon-delete" />  // use icon-trash
<i className="icon-close" />   // use icon-clear

// WRONG: Custom button component
const MyButton = () => <button>...</button>  // use BoostButton
```
