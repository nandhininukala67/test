@Path("/kycAdd")
@POST
@AuthPermission(value = {BaseService.MANAGE_PERMISSION})
public Response kycAdd(List<KycLimit> kycLimits) {
    Map<String, Object> resp = new LinkedHashMap<>();
    KycLimitManager kycLimitManager = getManager();

    for (KycLimit dtoLimit : kycLimits) {
        KycLimit existingLimit = kycLimitManager.getByID(dtoLimit.getId());

        // No existing record found
        if (existingLimit == null) {
            resp.put(JSON_KEY_MSG, "KYC Limit not found for ID: " + dtoLimit.getId());
            resp.put(JSON_KEY_SUCCESS, Boolean.FALSE);
            return Response.ok(resp, MediaType.APPLICATION_JSON).build();
        }

        // Check if changed
        if (!isChanged(existingLimit, dtoLimit)) {
            resp.put(JSON_KEY_MSG, "No changes detected.");
            resp.put(JSON_KEY_SUCCESS, Boolean.FALSE);
            return Response.ok(resp, MediaType.APPLICATION_JSON).build();
        }

        // Construct config key
        String kycConfig = "";
        if (existingLimit.getType() != null && !existingLimit.getType().isEmpty()) {
            kycConfig = "kyc." + existingLimit.getType().toLowerCase() + "." + existingLimit.getLimitType().toLowerCase() + ".";
        } else {
            resp.put(JSON_KEY_MSG, "KYC Format is Invalid");
            resp.put(JSON_KEY_SUCCESS, Boolean.FALSE);
            return Response.ok(resp, MediaType.APPLICATION_JSON).build();
        }

        // Validate
        if (validateRequest(dtoLimit, kycConfig, resp)) {
            return Response.ok(resp, MediaType.APPLICATION_JSON).build();
        }

        // Mark old record inactive
        existingLimit.setActive(false);
        getDB().saveOrUpdate(existingLimit);

        // Create new version of the record
        KycLimit newLimit = new KycLimit();
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
        newLimit.setPerTransaction(dtoLimit.getPerTransaction());
        newLimit.setMonthlyTrfOutwardCount(dtoLimit.getMonthlyTrfOutwardCount());
        newLimit.setCoolingLimit(dtoLimit.isCoolingLimit());
        newLimit.setActive(true);
        newLimit.setCreatedOn(new Date());
        // Copy any additional fields if required

        getDB().saveOrUpdate(newLimit);

        // Send to switch if NORMAL type
        if ("NORMAL".equalsIgnoreCase(newLimit.getLimitType())) {
            sendToRTSPSwitch(newLimit);
        }

        resp.put(JSON_KEY_MSG, "KYC Limit updated successfully.");
        resp.put(JSON_KEY_SUCCESS, Boolean.TRUE);
    }

    return Response.ok(resp, MediaType.APPLICATION_JSON).build();
}
