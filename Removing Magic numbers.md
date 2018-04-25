
# (Nearly) Removing Magic Numbers

## Background

Throughout the products, we us numbers to determine the outcome of an action. For example when we want to archive a Cost code, the stored procedure (changeCostcodeStatus) performs several checks to determine if the action can take place. These outcomes are returned from the stored
procedure in the format of a number.


```sql


BEGIN
	SET @count = (
			SELECT count(includeid)
			FROM signoffs
			WHERE [include] = 4
				AND includeid = @costcodeid
			)

	IF @count > 0
		RETURN - 1;

	SET @count = (
			SELECT count(costcodeid)
			FROM employee_costcodes
			WHERE costcodeid = @costcodeid
			)

	IF @count > 0
		RETURN - 2;
END


// Perform the action .... and return 0;

```


The response from the stored procedure is handeled in cCostsCode.ChangeStatus() which returns the respose from the stored procedure. 

cCostsCode.ChangeStatus() is called from a web method , which is called from costcode.js. The response of the web method is handled in the javascript

  ```javascript

  function changeStatusComplete(data) {
    switch (data) {
        case -1:
            SEL.MasterPopup.ShowMasterPopup(
                'This Cost Code cannot be archived as it is assigned to one or more Signoff Stages.',
                'Message from ' + moduleNameHTML);
            break;
        case -2:
            SEL.MasterPopup.ShowMasterPopup(
                'This Cost Code cannot be archived as it is currently assigned to one or more Employees.',
                'Message from ' + moduleNameHTML);
            break;
        default:
           //action ok so, update the UI 
    }
}
 
  ```

cCostsCode.ChangeStatus() can also be invoked from the CostCodeRepository, which is wired to Archive CostCode endpoint. Numerical response is handled in the repository with the outcome of the action be

```c#

  int outcome =_data.ChangeStatus(id, archive);
            switch (outcome)
            {
                case -1:
                    throw new ApiException(ApiResources.ApiErrorArchiveFailed,
                        ApiResources.ApiErrorCostCodeLinkedToSignoff);
                case -2:
                    throw new ApiException(ApiResources.ApiErrorArchiveFailed,
                        ApiResources.ApiErrorCostCodeLinkedToEmployee);            
            }

```

With the APIResources holding the API exception message returned to the caller

```c#
   public const string ApiErrorCostCodeLinkedToSignoff = "This Cost Code cannot be archived as it is assigned to one or more Signoff Stages.";
   public const string ApiErrorCostCodeLinkedToEmployee = "This Cost Code cannot be archived as it is currently assigned to one or more Employees.";

```




so what's the problem?

This presents several problems

* Cost code number is in multiple places, could be missed
* Maintain two error messages
* Not obvious what the numbers actually mean, without looking at their intereters in the JS and repo.
* 


## The solution

We really need cCostCode.ChangeStatus() method to return something more meaningful then an magic number, so why not return a descriptive outcome of the action, so the callers of the method don't have to
interpret what the integer means.

First thing is replace the magic numbers with an enum. These are mapped to the return values from stored procedure changeCostcodeStatus.

```c#

  public enum ChangeCostCodeArchiveStatus
    {
        /// <summary>
        /// Action was successfully.
        /// </summary>
        [Description("Success")]
        Success = 0,
        /// <summary>
        /// The Cost Code cannot be archived as it is assigned to one or more Signoff Stages.
        /// </summary>
        [Description("This Cost Code cannot be archived as it is assigned to one or more Signoff Stages.")]
        AssignedToSignOffStage = -1,
        /// <summary>
        /// The Cost Code cannot be archived as it is assigned to one or more Employees.
        /// </summary>
        [Description("This Cost Code cannot be archived as it is assigned to one or more Employees.")]
        AssignedToEmployee = -2,
    }

```

Note, there's a description attribute for each value, more on that later.

The return value from the stored procedure can then be cast to the ChangeCostCodeArchiveStatus enumerator


```c#
                int returnCode;
                  databaseConnection.sqlexecute.Parameters.AddWithValue("@delegateID", DBNull.Value);
                }
                databaseConnection.sqlexecute.Parameters.AddWithValue("@costcodeid", costCodeId);
                databaseConnection.sqlexecute.Parameters.AddWithValue("@archive", Convert.ToByte(archive));
                databaseConnection.sqlexecute.Parameters.Add("@returncode", SqlDbType.Int);
                databaseConnection.sqlexecute.Parameters["@returncode"].Direction = ParameterDirection.ReturnValue;
                databaseConnection.ExecuteProc("changeCostcodeStatus");
                returnCode = (int)databaseConnection.sqlexecute.Parameters["@returncode"].Value;
            }
            
       ChangeCostCodeArchiveStatus changeCostCodeStatusOutcome = (ChangeCostCodeArchiveStatus)returnCode;

```

Next, is to return the descriptive outcome. Using the following extention method we can read the [Des

