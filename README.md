@Path("/kycAdd")
@POST
@AuthPermission(value = {BaseService.MANAGE_PERMISSION})
public Response kycAdd(List<KycLimit> kycLimitList) {
    Map<String, Object> resp = new LinkedHashMap<>();
    KycLimitManager kycLimitManager = getManager();

    for (KycLimit dtoLimit : kycLimitList) {
        KycLimit existingLimit = kycLimitManager.getByID(dtoLimit.getId());

        if (existingLimit == null) {
            resp.put(JSON_KEY_MSG, "KYC Limit not found for ID: " + dtoLimit.getId());
            resp.put(JSON_KEY_SUCCESS, Boolean.FALSE);
            return Response.ok(resp, MediaType.APPLICATION_JSON).build();
        }

        if (!isChanged(existingLimit, dtoLimit)) {
            return error(Response.Status.NOT_MODIFIED, "Not Changed");
        }

        String kycConfig = "";
        if (isNotBlank(existingLimit.getType()) && isNotBlank(existingLimit.getLimitType())) {
            kycConfig = "kyc." + existingLimit.getType().toLowerCase() + "." + existingLimit.getLimitType().toLowerCase() + ".";
        } else {
            resp.put(JSON_KEY_MSG, "KYC Format is Invalid");
            resp.put(JSON_KEY_SUCCESS, Boolean.FALSE);
            return Response.ok(resp, MediaType.APPLICATION_JSON).build();
        }

        if (validateRequest(dtoLimit, kycConfig, resp)) {
            return Response.ok(resp, MediaType.APPLICATION_JSON).build();
        }

        // Step 1: Clone existing record as inactive
        KycLimit inactiveCopy = new KycLimit();
        inactiveCopy.setId(null); // Ensure new record
        inactiveCopy.setType(existingLimit.getType());
        inactiveCopy.setLimitType(existingLimit.getLimitType());
        inactiveCopy.setCapacityLimit(existingLimit.getCapacityLimit());
        inactiveCopy.setPerDayLoadLimit(existingLimit.getPerDayLoadLimit());
        inactiveCopy.setPerDayUnLoadLimit(existingLimit.getPerDayUnLoadLimit());
        inactiveCopy.setPerDayTrfInwardLimit(existingLimit.getPerDayTrfInwardLimit());
        inactiveCopy.setPerDayTfrOutwardLimit(existingLimit.getPerDayTfrOutwardLimit());
        inactiveCopy.setTxnLoadCount(existingLimit.getTxnLoadCount());
        inactiveCopy.setTxnLTfrInwardCount(existingLimit.getTxnLTfrInwardCount());
        inactiveCopy.setTxnUnloadCount(existingLimit.getTxnUnloadCount());
        inactiveCopy.setTxnTrfOutwardCount(existingLimit.getTxnTrfOutwardCount());
        inactiveCopy.setMonthlyTrfOutwardCount(existingLimit.getMonthlyTrfOutwardCount());
        inactiveCopy.setPerTransaction(existingLimit.getPerTransaction());
        inactiveCopy.setCoolingLimit(existingLimit.isCoolingLimit());
        inactiveCopy.setActive("inactive");

        getDB().saveOrUpdate(inactiveCopy);

        // Step 2: Save new incoming record as active
        dtoLimit.setId(null); // To insert as new record
        dtoLimit.setActive("active");
        getDB().saveOrUpdate(dtoLimit);

        // Step 3: Send to switch if normal type
        if (StringUtils.equals(dtoLimit.getLimitType(), KYC_LIMIT_TYPE_NORMAL)) {
            sendToRTSPSwitch(dtoLimit);
        }

        resp.put(JSON_KEY_MSG, "KYC Limit updated successfully.");
        resp.put(JSON_KEY_SUCCESS, Boolean.TRUE);
    }

    return Response.ok(resp, MediaType.APPLICATION_JSON).build();
}
