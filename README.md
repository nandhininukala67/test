@Path("/kycAdd")
@POST
@AuthPermission(value = {BaseService.MANAGE_PERMISSION})
public Response kycAdd(List<KycLimit> kycLimit) {
    Map<String, Object> resp = new LinkedHashMap<>();
    KycLimitManager kycLimitManager = getManager();

    for (KycLimit dtoLimit : kycLimit) {
        KycLimit existingLimit = kycLimitManager.getByID(dtoLimit.getId());
        if (!isChanged(existingLimit, dtoLimit)) {
            return error(Response.Status.NOT_MODIFIED, "Not Changed");
        }
    }

    for (KycLimit dtoLimit : kycLimit) {
        KycLimit existingLimit = kycLimitManager.getByID(dtoLimit.getId());
        String kycConfig = "";

        if (existingLimit != null && !existingLimit.getType().isEmpty()) {
            kycConfig = "kyc." + existingLimit.getType().toLowerCase() + "." + existingLimit.getLimitType().toLowerCase() + ".";
        } else {
            resp.put(JSON_KEY_MSG, "KYC Format is Invalid");
            resp.put(JSON_KEY_SUCCESS, Boolean.FALSE);
            return Response.ok(resp, MediaType.APPLICATION_JSON).build();
        }

        if (validateRequest(dtoLimit, kycConfig, resp)) {
            return Response.ok(resp, MediaType.APPLICATION_JSON).build();
        }

        // ✅ Archive the existing record
        KycLimitHistory history = new KycLimitHistory();
        history.setOriginalId(existingLimit.getId());
        history.setType(existingLimit.getType());
        history.setLimitType(existingLimit.getLimitType());
        history.setCapacityLimit(existingLimit.getCapacityLimit());
        history.setPerDayLoadLimit(existingLimit.getPerDayLoadLimit());
        history.setPerDayUnLoadLimit(existingLimit.getPerDayUnLoadLimit());
        history.setPerDayTrfInwardLimit(existingLimit.getPerDayTrfInwardLimit());
        history.setPerDayTfrOutwardLimit(existingLimit.getPerDayTfrOutwardLimit());
        history.setTxnLoadCount(existingLimit.getTxnLoadCount());
        history.setTxnLTfrInwardCount(existingLimit.getTxnLTfrInwardCount());
        history.setTxnUnloadCount(existingLimit.getTxnUnloadCount());
        history.setTxnTrfOutwardCount(existingLimit.getTxnTrfOutwardCount());
        history.setMonthlyTrfOutwardCount(existingLimit.getMonthlyTrfOutwardCount());
        history.setPerTransaction(existingLimit.getPerTransaction());
        history.setActive("N");
        history.setArchivedAt(LocalDateTime.now()); // ✅ capture current date and time

        // Save history
        getDB().saveOrUpdate(history);

        // ✅ Update the original entity
        existingLimit.setCapacityLimit(dtoLimit.getCapacityLimit());
        existingLimit.setPerDayLoadLimit(dtoLimit.getPerDayLoadLimit());
        existingLimit.setPerDayUnLoadLimit(dtoLimit.getPerDayUnLoadLimit());
        existingLimit.setPerDayTrfInwardLimit(dtoLimit.getPerDayTrfInwardLimit());
        existingLimit.setPerDayTfrOutwardLimit(dtoLimit.getPerDayTfrOutwardLimit());
        existingLimit.setTxnLoadCount(dtoLimit.getTxnLoadCount());
        existingLimit.setTxnLTfrInwardCount(dtoLimit.getTxnLTfrInwardCount());
        existingLimit.setTxnUnloadCount(dtoLimit.getTxnUnloadCount());
        existingLimit.setTxnTrfOutwardCount(dtoLimit.getTxnTrfOutwardCount());
        existingLimit.setPerTransaction(dtoLimit.getPerTransaction());
        existingLimit.setMonthlyTrfOutwardCount(dtoLimit.getMonthlyTrfOutwardCount());

        getDB().saveOrUpdate(existingLimit);

        // Send to switch (optional)
        if (StringUtils.equals(existingLimit.getLimitType(), KYC_LIMIT_TYPE_NORMAL)) {
            sendToRTSPSwitch(dtoLimit);
        }

        resp.put(JSON_KEY_MSG, "Kyc Limit Updated and Archived Successfully.");
        resp.put(JSON_KEY_SUCCESS, Boolean.TRUE);
    }

    return Response.ok(resp, MediaType.APPLICATION_JSON).build();
}























@Service
public class KycLimitHistoryService {

    @Autowired
    private KycLimitHistoryRepository kycLimitHistoryRepository;

    public void saveFromKycLimit(KycLimit kycLimit) {
        KycLimitHistory history = new KycLimitHistory();

        history.setOriginalId(kycLimit.getId());
        history.setType(kycLimit.getType());
        history.setCapacityLimit(kycLimit.getCapacityLimit());
        history.setPerDayLoadLimit(kycLimit.getPerDayLoadLimit());
        history.setPerDayUnLoadLimit(kycLimit.getPerDayUnLoadLimit());
        history.setPerDayTrfInwardLimit(kycLimit.getPerDayTrfInwardLimit());
        history.setPerDayTfrOutwardLimit(kycLimit.getPerDayTfrOutwardLimit());
        history.setTxnLoadCount(kycLimit.getTxnLoadCount());
        history.setTxnLTfrInwardCount(kycLimit.getTxnLTfrInwardCount());
        history.setTxnUnloadCount(kycLimit.getTxnUnloadCount());
        history.setTxnTrfOutwardCount(kycLimit.getTxnTrfOutwardCount());
        history.setCoolingLimit(kycLimit.isCoolingLimit());
        history.setPerTransaction(kycLimit.getPerTransaction());
        history.setMonthlyTrfOutwardLimit(kycLimit.getMonthlyTrfOutwardLimit());
        history.setMonthlyTrfOutwardCount(kycLimit.getMonthlyTrfOutwardCount());
        history.setLimitType(kycLimit.getLimitType());
        history.setActive("N"); // marking as inactive history
        history.setArchivedAt(LocalDateTime.now());

        kycLimitHistoryRepository.save(history);
    }
}




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














@Path("/system/kyclimit")
@AuthPermissionEntity("kyclimit")
public final class KycLimitService extends BaseService<KycLimit, KycLimit, KycLimitManager, Long> {
    private EntityManager entityManager;

    public KycLimitService() {
        super(KycLimit.class, KycLimitManager.class);
    }

    protected KycLimitService(Class<KycLimit> tClass, Class<KycLimitManager> kycLimitManagerClass) {
        super(tClass, kycLimitManagerClass);
    }

    protected boolean isChanged(KycLimit entity, KycLimit dtoData) {
        boolean equal = new EqualsBuilder()
                .append(entity.getId(), dtoData.getId())
                .append(entity.getType(), dtoData.getType())
                .append(entity.getCapacityLimit(), dtoData.getCapacityLimit())
                .append(entity.getPerDayLoadLimit(), dtoData.getPerDayLoadLimit())
                .append(entity.getPerDayUnLoadLimit(), dtoData.getPerDayUnLoadLimit())
                .append(entity.getPerDayTrfInwardLimit(), dtoData.getPerDayTrfInwardLimit())
                .append(entity.getPerDayTfrOutwardLimit(), dtoData.getPerDayTfrOutwardLimit())
                .append(entity.getTxnLoadCount(), dtoData.getTxnLoadCount())
                .append(entity.getTxnLTfrInwardCount(), dtoData.getTxnLTfrInwardCount())
                .append(entity.getTxnTrfOutwardCount(), dtoData.getTxnTrfOutwardCount())
                .append(entity.getTxnUnloadCount(), dtoData.getTxnUnloadCount())
                .append(entity.isCoolingLimit(), dtoData.isCoolingLimit())
                .append(entity.getPerTransaction(), dtoData.getPerTransaction())
                .append(entity.getMonthlyTrfOutwardLimit(), dtoData.getMonthlyTrfOutwardLimit())
                .append(entity.getMonthlyTrfOutwardCount(), dtoData.getMonthlyTrfOutwardCount())
                .isEquals();

        return !equal;
    }

    @Path("/kycAdd")
    @POST
    @AuthPermission(value = {BaseService.MANAGE_PERMISSION})
    public Response kycAdd(List<KycLimit> kycLimit) {
        Map<String, Object> resp = new LinkedHashMap<>();
        KycLimitManager kycLimitManager = getManager();

        for (KycLimit dtoLimit : kycLimit) {
            KycLimit limit = kycLimitManager.getByID(dtoLimit.getId());
            if (!isChanged(limit, dtoLimit)) {
                return error(Response.Status.NOT_MODIFIED, "Not Changed");
            }
        }

        for (KycLimit dtoLimit : kycLimit) {
            KycLimit limit = kycLimitManager.getByID(dtoLimit.getId());
            String kycConfig = "";

            if (limit != null && !limit.getType().isEmpty()) {
                kycConfig = "kyc." + limit.getType().toLowerCase() + "." + limit.getLimitType().toLowerCase() + ".";
            } else {
                resp.put(JSON_KEY_MSG, "KYC Format is Invalid");
                resp.put(JSON_KEY_SUCCESS, Boolean.FALSE);
                return Response.ok(resp, MediaType.APPLICATION_JSON).build();
            }

            if (validateRequest(dtoLimit, kycConfig, resp)) {
                return Response.ok(resp, MediaType.APPLICATION_JSON).build();
            }

            // Versioning: Mark old record as inactive (active = "n")
            limit.setActive("n");
            getDB().saveOrUpdate(limit);  // Save the old record as inactive

            // Create new record with updated values
            KycLimit newLimit = new KycLimit(dtoLimit);
            newLimit.setId(null);  // Generate a new ID for the new record
            newLimit.setActive("y");  // Set the new record as active
            newLimit.setCreatedAt(new Date());  // Set a new creation date (optional)
            getDB().saveOrUpdate(newLimit);

            // Now send this to Switch via a separate thread if required
            if (StringUtils.equals(limit.getLimitType(), KYC_LIMIT_TYPE_NORMAL)) {
                sendToRTSPSwitch(dtoLimit);
            }

            resp.put(JSON_KEY_MSG, "Kyc Limit Updated Successfully.");
            resp.put(JSON_KEY_SUCCESS, Boolean.TRUE);
        }

        return Response.ok(resp, MediaType.APPLICATION_JSON).build();
    }

    private static boolean validateRequest(KycLimit dtoLimit, String kycConfig, Map<String, Object> resp) {
        BigDecimal configLimitPerDayLoad = new BigDecimal(Config.getDBConfig(kycConfig + "per.day.load", "0"));
        BigDecimal perDayLoadLimit = dtoLimit.getPerDayLoadLimit();

        if (configLimitPerDayLoad.compareTo(perDayLoadLimit) < 0) {
            resp.put(JSON_KEY_MSG, "Per Day Load limit exceeded. </br>Configured limit is " + configLimitPerDayLoad);
            resp.put(JSON_KEY_SUCCESS, Boolean.FALSE);
            return true;
        }

        BigDecimal configLimitPerDayInward = new BigDecimal(Config.getDBConfig(kycConfig + "per.day.transfer.inward", "0"));
        BigDecimal perDayInwardLimit = dtoLimit.getPerDayTrfInwardLimit();

        if (configLimitPerDayInward.compareTo(perDayInwardLimit) < 0) {
            resp.put(JSON_KEY_MSG, "Per day transfer inward limit exceeded.</br>Configured limit is " + configLimitPerDayInward);
            resp.put(JSON_KEY_SUCCESS, Boolean.FALSE);
            return true;
        }

        BigDecimal configLimitPerDayUnLoad = new BigDecimal(Config.getDBConfig(kycConfig + "per.day.unload", "0"));
        BigDecimal perDayUnLoadLimit = dtoLimit.getPerDayUnLoadLimit();

        if (configLimitPerDayUnLoad.compareTo(perDayUnLoadLimit) < 0) {
            resp.put(JSON_KEY_MSG, "Per Day UnLoad limit exceeded. </br>Configured limit is " + configLimitPerDayUnLoad);
            resp.put(JSON_KEY_SUCCESS, Boolean.FALSE);
            return true;
        }

        BigDecimal configLimitPerDayOutward = new BigDecimal(Config.getDBConfig(kycConfig + "per.day.transfer.outward", "0"));
        BigDecimal perDayOutwardLimit = dtoLimit.getPerDayTfrOutwardLimit();

        if (configLimitPerDayOutward.compareTo(perDayOutwardLimit) < 0) {
            resp.put(JSON_KEY_MSG, "Per day outward limit exceeded.</br>Configured limit is " + configLimitPerDayOutward);
            resp.put(JSON_KEY_SUCCESS, Boolean.FALSE);
            return true;
        }

        long dbtxnLoadCount = Long.parseLong(Config.getDBConfig(kycConfig + "number.of.load.count", "0"));
        long txnLoadCount = dtoLimit.getTxnLoadCount();

        if (dbtxnLoadCount < txnLoadCount) {
            resp.put(JSON_KEY_MSG, "Number of load count limit exceeded. </br>Configured limit is " + dbtxnLoadCount);
            resp.put(JSON_KEY_SUCCESS, Boolean.FALSE);
            return true;
        }

        long dbtxnInwardCount = Long.parseLong(Config.getDBConfig(kycConfig + "number.of.inward.count", "0"));
        long txnInwardCount = dtoLimit.getTxnLTfrInwardCount();

        if (dbtxnInwardCount < txnInwardCount) {
            resp.put(JSON_KEY_MSG, "Number of inward txn count limit exceeded.</br>Configured limit is " + dbtxnInwardCount);
            resp.put(JSON_KEY_SUCCESS, Boolean.FALSE);
            return true;
        }

        long dbTxnUnloadCount = Long.parseLong(Config.getDBConfig(kycConfig + "number.of.unload.count", "0"));
        long txnUnloadCount = dtoLimit.getTxnUnloadCount();

        if (dbTxnUnloadCount < txnUnloadCount) {
            resp.put(JSON_KEY_MSG, "Number of Txn Unload count limit exceeded. </br>Configured limit is " + dbTxnUnloadCount);
            resp.put(JSON_KEY_SUCCESS, Boolean.FALSE);
            return true;
        }

        long dbTxnOutwardCount = Long.parseLong(Config.getDBConfig(kycConfig + "number.of.outward.count", "0"));
        long txnOutwardCount = dtoLimit.getTxnTrfOutwardCount();

        if (dbTxnOutwardCount < txnOutwardCount) {
            resp.put(JSON_KEY_MSG, "Number of outward txn count limit exceeded.</br>Configured limit is " + dbTxnOutwardCount);
            resp.put(JSON_KEY_SUCCESS, Boolean.FALSE);
            return true;
        }

        long dbMonthlyTrfOutwardCount = Long.parseLong(Config.getDBConfig(kycConfig + "monthly.trf.outward.count", "0"));
        long monthlyTrfOutwardCount = dtoLimit.getMonthlyTrfOutwardCount();

        if (dbMonthlyTrfOutwardCount < monthlyTrfOutwardCount) {
            resp.put(JSON_KEY_MSG, "Monthly transfer outward count limit exceeded.</br>Configured limit is " + dbMonthlyTrfOutwardCount);
            resp.put(JSON_KEY_SUCCESS, Boolean.FALSE);
            return true;
        }

        BigDecimal dbPerTxnLimit = new BigDecimal(Config.getDBConfig(kycConfig + "per.txn.limit", "0"));
        BigDecimal perTxnLimit = dtoLimit.getPerTransaction();

        if (dbPerTxnLimit.compareTo(perTxnLimit) < 0) {
            resp.put(JSON_KEY_MSG, "Per transaction limit exceeded.</br>Configured limit is " + dbPerTxnLimit);
            resp.put(JSON_KEY_SUCCESS, Boolean.FALSE);
            return true;
        }

        // Check wallet holding limit only for FULL KYC - NORMAL
        if (StringUtils.equals(kycConfig, "kyc.full.normal.")) {
            BigDecimal dbWalletHoldingLimit = new BigDecimal(Config.getDBConfig(kycConfig + "wallet.holding.limit", "0"));
            BigDecimal walletHoldingLimit = dtoLimit.getCapacityLimit();

            if (dbWalletHoldingLimit.compareTo(walletHoldingLimit) < 0) {
                resp.put(JSON_KEY_MSG, "Wallet holding limit exceeded.</br>Configured limit is " + dbWalletHoldingLimit);
                resp.put(JSON_KEY_SUCCESS, Boolean.FALSE);
                return true;
            }
        }

        return false;
    }

    private void sendToRTSPSwitch(KycLimit dtoLimit) {
        if (Config.getBoolean("post.kyc.capacity.limit.config.to.rtsp", true)) {
            Runnable worker = () -> {
                try {
                    String httpUrl = Config.getDBConfig(KYCLIMIT_URL_RTSP_SWITCH);
                    String httpsUrl = Config.getDBConfig(KYCLIMIT_HTTPS_URL_RTSP_SWITCH);
                    String channel = Config.getDBConfig(SWITCH_CHANNEL);
                    String institute = Config.getDBConfig(SWITCH_INSTITUTE);
                    String authorization = Config.getDBConfig(SWITCH_AUTHORIZATION);
                    String timeout = Config.getDBConfig(SWITCH_TIMEOUT);
                    String readTimeout = Config.getDBConfig(SWITCH_READ_TIMEOUT);
                    String https = Config.getDBConfig(SWITCH_PROTOCOL);
                    boolean isSecure = StringUtils.equals(SWITCH_HTTPS_PROTOCOL, https);

                    if (isNotBlank(httpUrl) && isNotBlank(channel) && isNotBlank(authorization)) {
                        JSONObject jsonPayload = new JSONObject();
                        jsonPayload.put(SWITCH_KYC_KEY, dtoLimit.getType());
                        jsonPayload.put(SWITCH_KYC_CAPACITY_LIMIT, dtoLimit.getCapacityLimit());

                        WebResource webResource = JerseyUtil.getInstance().getResource(Integer.valueOf(timeout), Integer.valueOf(readTimeout),
                                isSecure, jsonPayload, httpsUrl, httpUrl, getDB());

                        printSwitchApiTxns(SWITCH_RTSP, true, jsonPayload);

                        ClientResponse response = JerseyHelper.post(institute, channel, authorization, jsonPayload, webResource);
                        String stringifyResponse = response.getEntity(String.class);

                        printSwitchApiTxns(SWITCH_RTSP, false, stringifyResponse);
                    }
                } catch (Exception e) {
                    LOG.error(e.getMessage(), e);
                }
            };

            Thread thread = new Thread(worker);
            thread.start();
        }
    }
}

