# Good practices

Here is a list of convention our FT Optix technicians uses when creating new projects, feel free to choose if this can apply to your project

- Use significative names for every project element (no default names)
    - Exception is made when creating object types, still remember to give meaningful name if possible
    - Avoid using number at the end of item name
    - Avoid abbreviations in object names to improve readability
    - Examples:
        - `Templates/SpinBox (Type)` -> BAD
        - `Templates/SpinBoxWithGreenBorder (Type)` -> GOOD
        - `MainWindow/SpinBox1` -> BAD
        - `MainWindow/MotorSpeed_SpinBox` -> GOOD
        - `MainWindow/MotorSP_SB` -> BAD
- Nice to use same EmbeddedDatabase for every item to be stored
    - Multiple tables can be used in same EmbeddedDatabase with different settings (RecordLimit, Names, etc)
- Use `PascalCase` for object names
- Use `camelCase` for variable names (Model, Session and Local variables)
- All graphical elements should be in `UI` folder
	- Here you can create standard folders to organize your work, such as:
		- `UI/Templates`-> Contains all custom types used to create objects
		- `UI/Screens` -> Contains the different pages that are used in the project
	- Both subfolder can have children subfolder to organize elements if needed
		- Example:
			- `UI/Templates/MotorWidgets`
			- `UI/Templates/PumpsWidgets`
			- `UI/Screens/Filler`
			- `UI/Screens/Settings`
	- Nice to use Templates folder in any first level project folder (e.g. UI, Model, Converters, etc...) when creating custom types

- Global variables should be created in the `Model` folder (and subfolder)
- Nice to use HTML color palette if no special need for custom coloring
	- Example:
		- `red` -> GOOD
		- `#FFA1C8` -> BAD
- Object names and LocalizationDictionary keys should be in English
- Nice to sort items by their type in project tree
	- Exception is made when defining layers is necessary to build final interface (bottom most element is highest layer)
- When creating a new page/container, avoid using a `Panel` with a `Rectangle` as coloured background to avoid doubling project nodes
	- Use instead:
		- `Screen` if the object is a whole screen that is going to be displayed in the HMI
		- `Rectangle` if coloured background is needed
		- `Panel` if background should be transparent (usually to group elements)
- Nice to group items using Panel
	- This is useful to reduce project complexity when assigning groups-based permissions and/or to trigger specific visual logics (enabling, visibility, etc)
- Nice to use same `Label` name and `LocalizationDictionary` item key
- Nice to use keyword in controls names to identify which variable/object is referring to 
	- Example:
		- `VoltageText` -> Label that shows the translation key for the _Voltage_ element
		- `VoltageValue` -> Name for a LinearGauge controlling _Voltage_ value
