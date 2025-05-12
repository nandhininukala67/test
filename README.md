@Path("/kycAdd")
@POST
@AuthPermission(value = {BaseService.MANAGE_PERMISSION})
public Response kycAdd(List<KycLimit> kycLimitList) {
    List<Map<String, Object>> responseList = new ArrayList<>();
    KycLimitManager kycLimitManager = getManager();

    for (KycLimit dtoLimit : kycLimitList) {
        Map<String, Object> recordResp = new LinkedHashMap<>();
        recordResp.put("id", dtoLimit.getId());
        recordResp.put("type", dtoLimit.getType());
        recordResp.put("limitType", dtoLimit.getLimitType());

        KycLimit existingLimit = kycLimitManager.getByID(dtoLimit.getId());

        String kycConfig;
        if (dtoLimit.getType() != null && dtoLimit.getLimitType() != null) {
            kycConfig = "kyc." + dtoLimit.getType().toLowerCase() + "." + dtoLimit.getLimitType().toLowerCase() + ".";
        } else {
            recordResp.put(JSON_KEY_MSG, "KYC Format is Invalid");
            recordResp.put(JSON_KEY_SUCCESS, Boolean.FALSE);
            responseList.add(recordResp);
            continue;
        }

        if (validateRequest(dtoLimit, kycConfig, recordResp)) {
            responseList.add(recordResp);
            continue;
        }

        KycLimit newLimit;
        if (existingLimit != null) {
            // Step 1: Mark old record as inactive
            existingLimit.setStatus("inactive");
            getDB().saveOrUpdate(existingLimit);

            // Step 2: Clone old record into new one
            newLimit = new KycLimit();
            newLimit.setType(existingLimit.getType());
            newLimit.setLimitType(existingLimit.getLimitType());
        } else {
            // No existing record, create fresh
            newLimit = new KycLimit();
            newLimit.setType(dtoLimit.getType());
            newLimit.setLimitType(dtoLimit.getLimitType());
        }

        // Step 3: Apply updated values from request
        newLimit.setCapacityLimit(dtoLimit.getCapacityLimit());
        newLimit.setPerDayLoadLimit(dtoLimit.getPerDayLoadLimit());
        newLimit.setPerDayUnLoadLimit(dtoLimit.getPerDayUnLoadLimit());
        newLimit.setPerDayTrfInwardLimit(dtoLimit.getPerDayTrfInwardLimit());
        newLimit.setPerDayTfrOutwardLimit(dtoLimit.getPerDayTfrOutwardLimit());
        newLimit.setTxnLoadCount(dtoLimit.getTxnLoadCount());
        newLimit.setTxnLTfrInwardCount(dtoLimit.getTxnLTfrInwardCount());
        newLimit.setTxnUnloadCount(dtoLimit.getTxnUnloadCount());
        newLimit.setTxnTrfOutwardCount(dtoLimit.getTxnTrfOutwardCount());
        newLimit.setMonthlyTrfOutwardCount(dtoLimit.getMonthlyTrfOutwardCount());
        newLimit.setPerTransaction(dtoLimit.getPerTransaction());
        newLimit.setCoolingLimit(dtoLimit.isCoolingLimit());

        // Step 4: Mark new record as active
        newLimit.setStatus("active");

        // Save new record
        getDB().saveOrUpdate(newLimit);

        // Optional switch sync
        if (StringUtils.equals(newLimit.getLimitType(), KYC_LIMIT_TYPE_NORMAL)) {
            sendToRTSPSwitch(newLimit);
        }

        recordResp.put(JSON_KEY_MSG, "KYC Limit versioned and updated.");
        recordResp.put(JSON_KEY_SUCCESS, Boolean.TRUE);
        responseList.add(recordResp);
    }

    return Response.ok(responseList, MediaType.APPLICATION_JSON).build();
}




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

        // ✅ Step 1: Update existing record to inactive
        existingLimit.setActive("inactive");
        getDB().saveOrUpdate(existingLimit);
        getDB().flush(); // ✅ Flush to avoid constraint error during next insert

        // ✅ Step 2: Create and insert new active record
        KycLimit newLimit = new KycLimit();
        newLimit.setId(null); // Ensure it's a new record
        newLimit.setType(dtoLimit.getType());
        newLimit.setLimitType(dtoLimit.getLimitType());
        newLimit.setCapacityLimit(dtoLimit.getCapacityLimit());
        newLimit.setPerDayLoadLimit(dtoLimit.getPerDayLoadLimit());
        newLimit.setPerDayUnLoadLimit(dtoLimit.getPerDayUnLoadLimit());
        newLimit.setPerDayTrfInwardLimit(dtoLimit.getPerDayTrfInwardLimit());
        newLimit.setPerDayTfrOutwardLimit(dtoLimit.getPerDayTfrOutwardLimit());
        newLimit.setTxnLoadCount(dtoLimit.getTxnLoadCount());
        newLimit.setTxnLTfrInwardCount(dtoLimit.getTxnLTfrInwardCount());
        newLimit.setTxnUnloadCount(dtoLimit.getTxnUnloadCount());
        newLimit.setTxnTrfOutwardCount(dtoLimit.getTxnTrfOutwardCount());
        newLimit.setMonthlyTrfOutwardCount(dtoLimit.getMonthlyTrfOutwardCount());
        newLimit.setPerTransaction(dtoLimit.getPerTransaction());
        newLimit.setCoolingLimit(dtoLimit.isCoolingLimit());
        newLimit.setActive("active");

        getDB().saveOrUpdate(newLimit);

        // ✅ Step 3: Send to switch if required
        if (StringUtils.equalsIgnoreCase(newLimit.getLimitType(), KYC_LIMIT_TYPE_NORMAL)) {
            sendToRTSPSwitch(newLimit);
        }

        resp.put(JSON_KEY_MSG, "KYC Limit updated successfully.");
        resp.put(JSON_KEY_SUCCESS, Boolean.TRUE);
    }

    return Response.ok(resp, MediaType.APPLICATION_JSON).build();
}









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
