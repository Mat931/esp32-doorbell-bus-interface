## How to order

### JLCPCB (with assembly)

- Go to https://jlcpcb.com/quote/
- Upload the [Gerber file](https://github.com/Mat931/esp32-doorbell-bus-interface/tree/main/fabrication)
- Choose a color and thickness combination that is compatible with assembly (Recommended is green and 1.6mm, more info at https://jlcpcb.com/capabilities/pcb-assembly-capabilities)
- Select `Remove Order Number: Specify a location`
- Scroll down to `PCB Assembly` and turn on the switch
- Make sure `Economic` and `Top Side` are selected
- Select `Quantity (PCBA Qty): 2` or `5` depending on how many assembled boards you want to order
- Select `Tooling holes: Added by Customer`
- Click the blue `Confirm` button
- Click `Next`
- Upload the [BOM and CPL files](https://github.com/Mat931/esp32-doorbell-bus-interface/tree/main/fabrication)
- Click `Process BOM & CPL`
- If you get the message `P2 designator don't exist in the BOM file` click `Continue`
- Make sure every component in the list is in stock and has a check mark on the right
- If some components are out of stock you can replace them with similar parts
- Click `Next` twice
- Select a product description, for example `Research/Education/DIY/Entertainment` and `Development Board`
- Save to cart
