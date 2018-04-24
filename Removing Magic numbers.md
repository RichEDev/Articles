
# (Nearly) Removing Magic Numbers

Throughout the products, we us numbers to determine the outcome of an action. For example when we want to delete a Cost code, the stored procedure performs several checks to determine if the action can take place. These outcomes are returned from the stored
procedure in the format of a number.


```sql


	SET @count = (select count(*) from signoffs where include = 4 and includeid = @costcodeid)
	IF @count > 0
		RETURN -1;	
	SET @count = (select count (costcodeid) from employee_costcodes where costcodeid = @costcodeid)
	IF @count > 0
		RETURN -2;	
	SET @count = (select count (costcodeid) from savedexpenses_costcodes where costcodeid = @costcodeid)		
	IF @count > 0
		RETURN -4;
```


The response from the stored procedure is then processed ultimately be the Javascript, so we can return a more meaningful message to the user about the outcome of the action.

  ```javascript

  function deleteCostcodeComplete(data) {
    switch (data) {
        case -1:
            SEL.MasterPopup.ShowMasterPopup('This Cost Code cannot be deleted as it is assigned to one or more Signoff Stages.', 'Message from ' + moduleNameHTML);
            break;
        case -2:
            SEL.MasterPopup.ShowMasterPopup('This Cost Code cannot be deleted as it is assigned to one or more Employees.', 'Message from ' + moduleNameHTML);
            break;
        case -4:
            SEL.MasterPopup.ShowMasterPopup('This Cost Code cannot be deleted as it is assigned to one or more Expense Items.', 'Message from ' + moduleNameHTML);
            break;    
        default:
            SEL.Grid.deleteGridRow('gridCostcodes', currentRowID);
            break;
    }
}
 

  ```


This presents several problems

* Duplicate message for anyone calling from API
* Difficult to find all uses of the magic number

