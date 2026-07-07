Do validations and checks at the final destination unless needed for UX.
In the middle we do not care that the date is in the past - let the "owning" service decide if it wants to handle expired data or not.
In the middle do not check if all fields, headers are present for the call - let the "owning" service decide and just propagate the response
