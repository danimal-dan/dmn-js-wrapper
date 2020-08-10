# dmn-wrapper
Interface for Camunda DMN Tables. Should allow searching and modifying existing DMN
tables using a custom UI. Use `dmn-js` to display rendered result.

Try `dmn-moddle` for editing DMN XML files.

Try `faster-xml` for parsing XML into Javascript object to support search/filtering.
- Result: At this point, it looks like `dmn-moddle` can be used in place of this.

Try `xsd-schema-validator` for validating schema
- Result: This validator uses Java to perform the actual validation. So this would need to be done server-side.