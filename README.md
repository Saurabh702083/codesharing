package com.epay.admin.portal.service.master;

import com.epay.admin.portal.config.AdminPortalConfig;
import com.epay.admin.portal.dto.BaseMasterDto;
import com.epay.admin.portal.dto.master.MasterWorkFlowDto;
import com.epay.admin.portal.entity.admin.master.MasterBulkUploadConfig;
import com.epay.admin.portal.entity.admin.master.MasterBulkUploadFile;
import com.epay.admin.portal.entity.admin.master.MasterBulkUploadFileDetail;
import com.epay.admin.portal.exceptions.AdminPortalException;
import com.epay.admin.portal.externalservice.FileService;
import com.epay.admin.portal.model.request.master.*;
import com.epay.admin.portal.model.response.AdminPortalResponse;
import com.epay.admin.portal.model.response.master.MasterBulkUploadResponse;
import com.epay.admin.portal.util.AdminPortalUtil;
import com.epay.admin.portal.util.enums.*;
import com.epay.admin.portal.validator.MasterBulkUploadValidation;
import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.sbi.epay.logging.utility.LoggerFactoryUtility;
import com.sbi.epay.logging.utility.LoggerUtility;
import jakarta.servlet.http.HttpServletResponse;
import lombok.RequiredArgsConstructor;
import org.apache.commons.lang3.StringUtils;
import org.slf4j.MDC;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.web.multipart.MultipartFile;

import java.text.MessageFormat;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.Map;
import java.util.Optional;

import static com.epay.admin.portal.util.AdminPortalConstants.*;
import static com.epay.admin.portal.util.AdminPortalUtil.*;

/**
 * Class Name: AdminMasterService
 * *
 * Author: V1018841(Saurabh Mahto)
 * <p>
 * Copyright (c) 2026 [State Bank of India]
 * All rights reserved
 * *
 * Version:1.0
 */
@Service
@RequiredArgsConstructor
public class AdminMasterService {
    private final LoggerUtility logger = LoggerFactoryUtility.getLogger(this.getClass());
    private final MasterHandlerFactory<BaseMasterDto> masterHandlerFactory;
    private final MasterWorkFlowService masterWorkFlowService;
    private final MasterBulkUploadService masterBulkUploadService;
    private final FileService fileService;
    private final AdminPortalConfig adminPortalConfig;
    private final MasterBulkUploadValidation bulkUploadValidation;
    private final ObjectMapper objectMapper;

    @Transactional
    public AdminPortalResponse<String> create(MasterRequest masterRequest, String masterCode) {
        BaseMasterService<BaseMasterDto> masterService = resolveMasterService(masterCode);

        BaseMasterDto masterDto = masterService.mapToMasterDto(masterRequest);
        logger.info("Create master request :{}", masterDto);

        masterService.validate(masterDto, WorkflowOperation.CREATE);
        logger.info("Request validated successfully.");

        masterDto = masterService.save(masterDto, WorkflowOperation.CREATE);
        logger.info("Record Created successfully for master code: {}", masterDto.getCode());

        MasterWorkFlowDto masterWorkFlowDto = masterService.mapToMasterWorkFlowDto(masterDto, WorkflowOperation.CREATE, MessageFormat.format(MASTER_ACTION_PERFORMED, WorkflowOperation.CREATE.name()));
        masterWorkFlowService.save(masterWorkFlowDto);
        logger.info("Master flow record updated for :{}", masterDto.getCode());

        return mapToResponse(MessageFormat.format(MASTER_ACTION_PERFORMED, WorkflowOperation.CREATE.name()));
    }

    @Transactional
    public AdminPortalResponse<String> update(MasterRequest masterRequest, String masterCode) {
        BaseMasterService<BaseMasterDto> masterService = resolveMasterService(masterCode);

        BaseMasterDto masterDto = masterService.mapToMasterDto(masterRequest);
        logger.info("Create master request :{}", masterDto);

        masterService.validate(masterDto, WorkflowOperation.UPDATE);
        masterDto = masterService.save(masterDto, WorkflowOperation.UPDATE);
        logger.info("Record update successfully for master code: {}", masterDto.getCode());

        MasterWorkFlowDto masterWorkFlowDto = masterService.mapToMasterWorkFlowDto(masterDto, WorkflowOperation.UPDATE, MessageFormat.format(MASTER_ACTION_PERFORMED, WorkflowOperation.UPDATE.name()));
        BaseMasterDto savedMasterDto = masterService.findByIds(List.of(masterDto.getId())).getFirst();
        masterWorkFlowService.update(masterWorkFlowDto, savedMasterDto, masterDto);
        logger.info("Master flow record updated for :{}", masterDto.getCode());

        return mapToResponse(MessageFormat.format(MASTER_ACTION_PERFORMED, WorkflowOperation.UPDATE.name()));
    }

    @Transactional(readOnly = true)
    public AdminPortalResponse<Map<String, Object>> search(String masterCode, MasterSearchRequest request, Pageable pageable) {
        BaseMasterService<BaseMasterDto> masterService = resolveMasterService(masterCode);
        Page<BaseMasterDto> page = masterService.search(request, pageable);
        return masterService.mapToResponse(page.getContent(), page.getTotalElements());
    }

    @Transactional
    public AdminPortalResponse<String> approve(MasterBulkRequest request, String masterCode) {
        return performAction(request, WorkflowOperation.APPROVE, List.of(MasterAction.PENDING_FOR_APPROVAL.name(), MasterAction.PENDING_FOR_INACTIVE.name(),  MasterAction.PENDING_FOR_REJECTED.name()), masterCode);
    }

    @Transactional
    public AdminPortalResponse<String> reject(MasterBulkRequest request, String masterCode) {
        return performAction(request, WorkflowOperation.REJECT, List.of(MasterAction.PENDING_FOR_APPROVAL.name(), MasterAction.PENDING_FOR_INACTIVE.name()), masterCode);
    }

    @Transactional
    public AdminPortalResponse<String> inactive(MasterBulkRequest request, String masterCode) {
        return performAction(request, WorkflowOperation.INACTIVE, List.of(MasterAction.APPROVED.name()), masterCode);
    }

    private AdminPortalResponse<String> performAction(MasterBulkRequest request, WorkflowOperation performOperation, List<String> allowedPreviousActions, String masterCode) {
        BaseMasterService<BaseMasterDto> masterService = resolveMasterService(masterCode);
        masterService.validateAction(request, performOperation.name());

        List<BaseMasterDto> masters = masterService.findByIds(request.getMasterIds());
        masterService.validateBulkMasters(masters, allowedPreviousActions, performOperation.name());
        logger.info("Request validated successfully for operation : {}.", performOperation.name());

        masterService.performAction(request.getMasterIds(), performOperation);
        logger.info(" {} : Operation performed successfully for master code: {}", performOperation.name(), masterCode);

        String remarks = StringUtils.isNotEmpty(request.getRemarks()) ? request.getRemarks() : MessageFormat.format(MASTER_ACTION_PERFORMED, performOperation.name());
        List<MasterWorkFlowDto> masterWorkFlowDtos = masters.stream().map(master -> masterService.mapToMasterWorkFlowDto(master, performOperation, remarks)).toList();
        masterWorkFlowService.save(masterWorkFlowDtos);
        logger.info("Master flow record saved of :{}, for Operation : {}", masterCode, performOperation.name());

        return mapToResponse(MessageFormat.format(MASTER_ACTION_PERFORMED, performOperation.name()));
    }


    private AdminPortalResponse<String> mapToResponse(String s) {
        return AdminPortalResponse.<String>builder().status(SUCCESS_RESPONSE_CODE).data(List.of(s)).count(1L).total(1L).build();
    }

    /**
     * Bulk uploads master data via a CSV file: validates and stores the file,
     * parses it into records, processes each record (create or inactivate,
     * depending on its status column), and persists per-row and overall
     * upload results.
     *
     * @param masterCode String
     * @param file       multiPartFile
     * @return MasterBulkUploadResponse
     */
    public AdminPortalResponse<MasterBulkUploadResponse> upload(String masterCode, MultipartFile file) {
        logger.info("Starting bulk upload for masterCode: {}, fileName: {}", masterCode, file.getOriginalFilename());
        BaseMasterService<BaseMasterDto> masterService = resolveMasterService(masterCode);
        MasterBulkUploadConfig bulkUploadConfig = masterBulkUploadService.getBulkUploadConfig(masterCode);
        // validate file if csv or not
        String checksum = AdminPortalUtil.generateChecksum(AdminPortalUtil.readFileBytes(file));
        bulkUploadValidation.validateFile(file, bulkUploadConfig, checksum);
        List<String> headers = readHeaders(bulkUploadConfig.getFileFormatConfig());
        // Upload file on s3
        String filePath = fileService.uploadFile(file);
        //insert file details in masterBulkUploadFile Table.
        MasterBulkUploadFile uploadFile = masterBulkUploadService.saveBulkUploadFile(masterCode, file, filePath, bulkUploadConfig.getId(), checksum);
        try {
            List<Map<String, String>> records = masterBulkUploadService.readRecords(file, headers);
            processUploadRecords(masterCode, masterService, uploadFile, records);
            logger.info("Bulk upload finished successfully for masterCode: {}, uploadFileId: {}", masterCode, uploadFile.getId());
            return buildUploadResponse(uploadFile);
        } catch (AdminPortalException  ex) {
            logger.error("Bulk upload failed for masterCode: {}, uploadFileId: {}. Reason: {}", masterCode, uploadFile.getId(), ex.getMessage());
            markUploadAsFailed(uploadFile, ex.getMessage());
            throw ex;
        }
    }

    private List<String> readHeaders(String fileFormatConfig) {
        try {
            return objectMapper.readValue(fileFormatConfig, new TypeReference<List<String>>() {});
        } catch (Exception exception) {
            throw new AdminPortalException(FAILED_ERROR_CODE, "Invalid bulk upload header configuration.");
        }
    }

    /**
     * Marks the given upload-file record as FAILED and persists a truncated remark (prefixed with the current correlation id) explaining the failure.
     *
     * @param uploadFile the upload-file record to mark as failed
     * @param remark     the failure reason
     */
    private void markUploadAsFailed(MasterBulkUploadFile uploadFile, String remark) {
        uploadFile.setParsingStatus(BulkUploadParsingStatus.FAILED);
        uploadFile.setFileStatus(BulkUploadFileStatus.FAILED);
        String remarkMessage= MDC.get(CORRELATION_ID)+"||"+StringUtils.truncate(remark,800);
        uploadFile.setRemark(StringUtils.defaultIfBlank(remarkMessage, "Bulk upload failed."));
        masterBulkUploadService.updateFileRecord(uploadFile);
        logger.info("Upload file id: {} marked as FAILED", uploadFile.getId());
    }

    private BaseMasterService<BaseMasterDto> resolveMasterService(String masterCode) {
        MasterType masterType = MasterType.getMasterType(masterCode);
        logger.info("Master request for: {}", masterType);
        return masterHandlerFactory.getMasterService(masterType.getCode());
    }

    /**
     * Processes all parsed bulk-upload records in batches: for each batch, applies the create/inactivate logic per row, tallies success/failure counts, persists
     * workflow entries and per-row detail records, then updates the overall upload file record with the final counts and status.
     *
     * @param masterCode   the master type code
     * @param masterService the resolved master service handler
     * @param uploadFile   the upload-file record being processed
     * @param records      the parsed CSV records
     */
    private void processUploadRecords(String masterCode, BaseMasterService<BaseMasterDto> masterService, MasterBulkUploadFile uploadFile, List<Map<String, String>> records) {
        long successCount = 0;
        long failedCount = 0;
        int batchSize = adminPortalConfig.getMasterBulkUploadBatchSize();
        logger.info("Processing {} record(s) for masterCode: {} in batches of {}", records.size(), masterCode, batchSize);
        for (int start = 0; start < records.size(); start += batchSize) {
            int end = Math.min(start + batchSize, records.size());
            List<Map<String, String>> batchRecords = records.subList(start, end);
            BulkUploadProcessResult processResult = processRecords(masterCode, masterService, batchRecords);
            List<BulkUploadRowResult> rowResults = processResult.rowResults();
            for (int index = 0; index < batchRecords.size(); index++) {
                BulkUploadRowResult rowResult = rowResults.get(index);
                if (BulkUploadRecordStatus.SUCCESS.name().equals(rowResult.recordStatus())) {
                    successCount++;
                } else {
                    failedCount++;
                }
            }
            if (processResult.workflowDtos() != null && !processResult.workflowDtos().isEmpty()) {
                masterWorkFlowService.save(processResult.workflowDtos());
            }
            masterBulkUploadService.saveMasterFileDtls(buildUploadFileDetails(uploadFile, batchRecords, rowResults));
        }
        updateMasterFile(uploadFile, records, successCount, failedCount);
        logger.info("Bulk upload completed for master code: {}. total: {}, success: {}, failed: {}", masterCode, records.size(), successCount, failedCount);
    }


    /**
     * Updates the upload-file record with final total/success/failed counts and sets its parsing/file status accordingly.
     *
     * @param uploadFile   the upload-file record to update
     * @param records      the full set of parsed records
     * @param successCount number of successfully processed records
     * @param failedCount  number of failed records
     */
    private void updateMasterFile(MasterBulkUploadFile uploadFile, List<Map<String, String>> records, long successCount, long failedCount) {
        uploadFile.setTotalRecords((long) records.size());
        uploadFile.setSuccessRecords(successCount);
        uploadFile.setFailedRecords(failedCount);
        uploadFile.setParsingStatus(BulkUploadParsingStatus.COMPLETED);
        uploadFile.setFileStatus(failedCount > 0 ? BulkUploadFileStatus.COMPLETED_WITH_ERRORS : BulkUploadFileStatus.COMPLETED);
        uploadFile.setRemark(String.format("Bulk upload processed. Success records: %d, Failed records: %d", successCount, failedCount));
        masterBulkUploadService.updateFileRecord(uploadFile);
        logger.info("Upload file id: {} updated with success: {}, failed: {}, status: {}", uploadFile.getId(), successCount, failedCount, uploadFile.getFileStatus());
    }

    private AdminPortalResponse<MasterBulkUploadResponse> buildUploadResponse(MasterBulkUploadFile uploadFile) {
        MasterBulkUploadResponse data = MasterBulkUploadResponse.builder().uploadFileId(uploadFile.getId()).masterType(uploadFile.getMasterType()).totalRecords(uploadFile.getTotalRecords()).successRecords(uploadFile.getSuccessRecords()).failedRecords(uploadFile.getFailedRecords()).parsingStatus(String.valueOf(uploadFile.getParsingStatus())).fileStatus(String.valueOf(uploadFile.getFileStatus())).remark(uploadFile.getRemark()).build();
        return AdminPortalResponse.<MasterBulkUploadResponse>builder().status(SUCCESS_RESPONSE_CODE).data(List.of(data)).total(1L).build();
    }

    /**
     * Processes a single batch of records for a given master handler, converting any unexpected {@link AdminPortalException} into a failure result for every
     * row in the batch rather than aborting the whole upload.
     *
     * @param masterType  the master type code
     * @param handler     the resolved master service handler
     * @param records     the batch of raw records
     * @return the batch's row results and any workflow entries to persist
     */
    private BulkUploadProcessResult processRecords(String masterType, BaseMasterService<BaseMasterDto> handler,  List<Map<String, String>> records) {
        try {
            List<BulkUploadRowResult> rowResults = new ArrayList<>(Collections.nCopies(records.size(), null));
            List<MasterWorkFlowDto> workflowDtos = new ArrayList<>();
            List<CreateMaster> createMasters = new ArrayList<>();
            List<CreateInactivation> createInactivation = new ArrayList<>();

            for (int index = 0; index < records.size(); index++) {
                Map<String, String> record = records.get(index);
                BaseMasterDto dto = toDto(record, handler.getDtoTargetClass());
                String status = record.get(STATUS);

                try {
                    if (StringUtils.isBlank(status) || ACTIVE.equalsIgnoreCase(status)) {
                        handler.validate(dto, WorkflowOperation.CREATE);
                        if (handler.findExisting(dto).isPresent()) {
                            rowResults.set(index, failureResult(RECORD_ALREADY_EXISTS));
                        } else {
                            createMasters.add(new CreateMaster(index, dto));
                        }
                        continue;
                    }

                    if (INACTIVE.equalsIgnoreCase(status)) {
                        Optional<BaseMasterDto> activeRecord = handler.findActiveApproved(dto);
                        if (activeRecord.isPresent()) {
                            createInactivation.add(new CreateInactivation(index, activeRecord.get()));
                        } else {
                            rowResults.set(index, failureResult(NO_VALID_RECORDS));
                        }
                        continue;
                    }

                    rowResults.set(index, failureResult("Unsupported status value: " + status));
                } catch (AdminPortalException ex) {
                    logger.info("Exception occurred while processing record: {}, error: {}", dto, ex.getMessage());
                    rowResults.set(index, failureResult(ex.getMessage()));
                }
            }

            createMaster(handler, createMasters, rowResults, workflowDtos);
            createInactivation(handler, createInactivation, rowResults, workflowDtos);
            return new BulkUploadProcessResult(rowResults, workflowDtos);
        } catch (AdminPortalException ex) {
            logger.error("Batch processing failed for masterType: {}. Reason: {}", masterType, ex.getMessage());
            return new BulkUploadProcessResult(records.stream().map(context -> failureResult(ex.getMessage())).toList(), List.of());
        }
    }

    private void createMaster(BaseMasterService<BaseMasterDto> handler, List<CreateMaster> createMasters, List<BulkUploadRowResult> rowResults, List<MasterWorkFlowDto> workflowDtos) {
        if (createMasters.isEmpty()) {
            return;
        }
        List<BaseMasterDto> dtos = createMasters.stream().map(CreateMaster::dto).toList();
        List<BaseMasterDto> savedDtos = handler.saveAll(dtos);
        for (int i = 0; i < createMasters.size(); i++) {
            CreateMaster createMaster = createMasters.get(i);
            BaseMasterDto savedDto = savedDtos.get(i);
            workflowDtos.add(handler.mapToMasterWorkFlowDto(savedDto, WorkflowOperation.CREATE, MessageFormat.format("Bulk upload {0} action performed.", WorkflowOperation.CREATE.name())));
            rowResults.set(createMaster.index(), new BulkUploadRowResult(BulkUploadRecordStatus.SUCCESS.name(), WorkflowOperation.CREATE.name(), CREATE_PENDING_REMARK, savedDto.getId()));
        }
    }

    private void createInactivation(BaseMasterService<BaseMasterDto> handler, List<CreateInactivation> createInactivations, List<BulkUploadRowResult> rowResults, List<MasterWorkFlowDto> workflowDtos) {
        if (createInactivations.isEmpty()) {
            return;
        }
        handler.performAction(createInactivations.stream().map(createInactivation -> createInactivation.record().getId()).toList(), WorkflowOperation.INACTIVE);
        for (CreateInactivation createInactivation : createInactivations) {
            BaseMasterDto record = createInactivation.record();
            workflowDtos.add(handler.mapToMasterWorkFlowDto(record, WorkflowOperation.INACTIVE, MessageFormat.format("Bulk upload {0} action performed.", WorkflowOperation.INACTIVE.name())));
            rowResults.set(createInactivation.index(), new BulkUploadRowResult(BulkUploadRecordStatus.SUCCESS.name(), WorkflowOperation.INACTIVE.name(), INACTIVE_PENDING_REMARK, record.getId()));
        }
    }

    private BulkUploadRowResult failureResult(String message) {
        String remarkMessage= MDC.get(CORRELATION_ID)+"||"+StringUtils.truncate(message,800);
        return new BulkUploadRowResult(BulkUploadRecordStatus.FAILED.name(), null, remarkMessage, null);
    }



    private List<MasterBulkUploadFileDetail> buildUploadFileDetails(MasterBulkUploadFile uploadFile, List<Map<String, String>> records, List<BulkUploadRowResult> rowResults) {
        List<MasterBulkUploadFileDetail> fileDetails = new ArrayList<>(records.size());
        for (int index = 0; index < records.size(); index++) {
            fileDetails.add(mapDetail(uploadFile, records.get(index), rowResults.get(index)));
        }
        return fileDetails;
    }

    private MasterBulkUploadFileDetail mapDetail(MasterBulkUploadFile uploadFile, Map<String, String> record, BulkUploadRowResult rowResult) {
        MasterBulkUploadFileDetail detail = new MasterBulkUploadFileDetail();
        detail.setMasterUploadFileId(uploadFile.getId());
        detail.setRecordJson(toJson(record));
        detail.setStatus(rowResult.recordStatus());
        detail.setOperation(rowResult.operation());
        detail.setRemark(rowResult.remark());
        detail.setMasterRecordId(rowResult.masterRecordId());
        return detail;
    }

    private record CreateMaster(int index, BaseMasterDto dto) {
    }

    private record CreateInactivation(int index, BaseMasterDto record) {
    }

    private record BulkUploadRowResult(String recordStatus, String operation, String remark, java.util.UUID masterRecordId) {
    }

    private record BulkUploadProcessResult(List<BulkUploadRowResult> rowResults, List<MasterWorkFlowDto> workflowDtos) {
    }

    /**
     * This is to download the csv file based upon the status for the bulk upload data.
     * @param response HttpServletResponse
     * @param bulkDownloadRequest bulk download request object
     */
    public void downloadBulkMaster(HttpServletResponse response,BulkDownloadRequest bulkDownloadRequest) {
        bulkUploadValidation.validateDownloadBulkMaster(bulkDownloadRequest.getBulkId(),bulkDownloadRequest.getBulkStatus(), bulkDownloadRequest.getMasterCode());
        masterBulkUploadService.downloadBulkUploadMaster(response, bulkDownloadRequest.getBulkId(),bulkDownloadRequest.getBulkStatus(), bulkDownloadRequest.getMasterCode());
    }
}

=================================
package com.epay.admin.portal.service.master;

import com.epay.admin.portal.config.AdminPortalConfig;
import com.epay.admin.portal.dto.BaseMasterDto;
import com.epay.admin.portal.dto.master.MasterWorkFlowDto;
import com.epay.admin.portal.entity.admin.master.MasterBulkUploadConfig;
import com.epay.admin.portal.entity.admin.master.MasterBulkUploadFile;
import com.epay.admin.portal.entity.admin.master.MasterBulkUploadFileDetail;
import com.epay.admin.portal.exceptions.AdminPortalException;
import com.epay.admin.portal.externalservice.FileService;
import com.epay.admin.portal.model.request.master.*;
import com.epay.admin.portal.model.response.AdminPortalResponse;
import com.epay.admin.portal.model.response.master.MasterBulkUploadResponse;
import com.epay.admin.portal.util.AdminPortalUtil;
import com.epay.admin.portal.util.enums.*;
import com.epay.admin.portal.validator.MasterBulkUploadValidation;
import com.fasterxml.jackson.core.type.TypeReference;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.sbi.epay.logging.utility.LoggerFactoryUtility;
import com.sbi.epay.logging.utility.LoggerUtility;
import jakarta.servlet.http.HttpServletResponse;
import lombok.RequiredArgsConstructor;
import org.apache.commons.lang3.StringUtils;
import org.slf4j.MDC;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.web.multipart.MultipartFile;

import java.text.MessageFormat;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.Map;
import java.util.Optional;

import static com.epay.admin.portal.util.AdminPortalConstants.*;
import static com.epay.admin.portal.util.AdminPortalUtil.*;

/**
 * Class Name: AdminMasterService
 * *
 * Author: V1018841(Saurabh Mahto)
 * <p>
 * Copyright (c) 2026 [State Bank of India]
 * All rights reserved
 * *
 * Version:1.0
 */
@Service
@RequiredArgsConstructor
public class AdminMasterService {
    private final LoggerUtility logger = LoggerFactoryUtility.getLogger(this.getClass());
    private final MasterHandlerFactory<BaseMasterDto> masterHandlerFactory;
    private final MasterWorkFlowService masterWorkFlowService;
    private final MasterBulkUploadService masterBulkUploadService;
    private final FileService fileService;
    private final AdminPortalConfig adminPortalConfig;
    private final MasterBulkUploadValidation bulkUploadValidation;
    private final ObjectMapper objectMapper;

    @Transactional
    public AdminPortalResponse<String> create(MasterRequest masterRequest, String masterCode) {
        BaseMasterService<BaseMasterDto> masterService = resolveMasterService(masterCode);

        BaseMasterDto masterDto = masterService.mapToMasterDto(masterRequest);
        logger.info("Create master request :{}", masterDto);

        masterService.validate(masterDto, WorkflowOperation.CREATE);
        logger.info("Request validated successfully.");

        masterDto = masterService.save(masterDto, WorkflowOperation.CREATE);
        logger.info("Record Created successfully for master code: {}", masterDto.getCode());

        MasterWorkFlowDto masterWorkFlowDto = masterService.mapToMasterWorkFlowDto(masterDto, WorkflowOperation.CREATE, MessageFormat.format(MASTER_ACTION_PERFORMED, WorkflowOperation.CREATE.name()));
        masterWorkFlowService.save(masterWorkFlowDto);
        logger.info("Master flow record updated for :{}", masterDto.getCode());

        return mapToResponse(MessageFormat.format(MASTER_ACTION_PERFORMED, WorkflowOperation.CREATE.name()));
    }

    @Transactional
    public AdminPortalResponse<String> update(MasterRequest masterRequest, String masterCode) {
        BaseMasterService<BaseMasterDto> masterService = resolveMasterService(masterCode);

        BaseMasterDto masterDto = masterService.mapToMasterDto(masterRequest);
        logger.info("Create master request :{}", masterDto);

        masterService.validate(masterDto, WorkflowOperation.UPDATE);
        masterDto = masterService.save(masterDto, WorkflowOperation.UPDATE);
        logger.info("Record update successfully for master code: {}", masterDto.getCode());

        MasterWorkFlowDto masterWorkFlowDto = masterService.mapToMasterWorkFlowDto(masterDto, WorkflowOperation.UPDATE, MessageFormat.format(MASTER_ACTION_PERFORMED, WorkflowOperation.UPDATE.name()));
        BaseMasterDto savedMasterDto = masterService.findByIds(List.of(masterDto.getId())).getFirst();
        masterWorkFlowService.update(masterWorkFlowDto, savedMasterDto, masterDto);
        logger.info("Master flow record updated for :{}", masterDto.getCode());

        return mapToResponse(MessageFormat.format(MASTER_ACTION_PERFORMED, WorkflowOperation.UPDATE.name()));
    }

    @Transactional(readOnly = true)
    public AdminPortalResponse<Map<String, Object>> search(String masterCode, MasterSearchRequest request, Pageable pageable) {
        BaseMasterService<BaseMasterDto> masterService = resolveMasterService(masterCode);
        Page<BaseMasterDto> page = masterService.search(request, pageable);
        return masterService.mapToResponse(page.getContent(), page.getTotalElements());
    }

    @Transactional
    public AdminPortalResponse<String> approve(MasterBulkRequest request, String masterCode) {
        return performAction(request, WorkflowOperation.APPROVE, List.of(MasterAction.PENDING_FOR_APPROVAL.name(), MasterAction.PENDING_FOR_INACTIVE.name(),  MasterAction.PENDING_FOR_REJECTED.name()), masterCode);
    }

    @Transactional
    public AdminPortalResponse<String> reject(MasterBulkRequest request, String masterCode) {
        return performAction(request, WorkflowOperation.REJECT, List.of(MasterAction.PENDING_FOR_APPROVAL.name(), MasterAction.PENDING_FOR_INACTIVE.name()), masterCode);
    }

    @Transactional
    public AdminPortalResponse<String> inactive(MasterBulkRequest request, String masterCode) {
        return performAction(request, WorkflowOperation.INACTIVE, List.of(MasterAction.APPROVED.name()), masterCode);
    }

    private AdminPortalResponse<String> performAction(MasterBulkRequest request, WorkflowOperation performOperation, List<String> allowedPreviousActions, String masterCode) {
        BaseMasterService<BaseMasterDto> masterService = resolveMasterService(masterCode);
        masterService.validateAction(request, performOperation.name());

        List<BaseMasterDto> masters = masterService.findByIds(request.getMasterIds());
        masterService.validateBulkMasters(masters, allowedPreviousActions, performOperation.name());
        logger.info("Request validated successfully for operation : {}.", performOperation.name());

        masterService.performAction(request.getMasterIds(), performOperation);
        logger.info(" {} : Operation performed successfully for master code: {}", performOperation.name(), masterCode);

        String remarks = StringUtils.isNotEmpty(request.getRemarks()) ? request.getRemarks() : MessageFormat.format(MASTER_ACTION_PERFORMED, performOperation.name());
        List<MasterWorkFlowDto> masterWorkFlowDtos = masters.stream().map(master -> masterService.mapToMasterWorkFlowDto(master, performOperation, remarks)).toList();
        masterWorkFlowService.save(masterWorkFlowDtos);
        logger.info("Master flow record saved of :{}, for Operation : {}", masterCode, performOperation.name());

        return mapToResponse(MessageFormat.format(MASTER_ACTION_PERFORMED, performOperation.name()));
    }


    private AdminPortalResponse<String> mapToResponse(String s) {
        return AdminPortalResponse.<String>builder().status(SUCCESS_RESPONSE_CODE).data(List.of(s)).count(1L).total(1L).build();
    }

    /**
     * Bulk uploads master data via a CSV file: validates and stores the file,
     * parses it into records, processes each record (create or inactivate,
     * depending on its status column), and persists per-row and overall
     * upload results.
     *
     * @param masterCode String
     * @param file       multiPartFile
     * @return MasterBulkUploadResponse
     */
    public AdminPortalResponse<MasterBulkUploadResponse> upload(String masterCode, MultipartFile file) {
        logger.info("Starting bulk upload for masterCode: {}, fileName: {}", masterCode, file.getOriginalFilename());
        BaseMasterService<BaseMasterDto> masterService = resolveMasterService(masterCode);
        MasterBulkUploadConfig bulkUploadConfig = masterBulkUploadService.getBulkUploadConfig(masterCode);
        // validate file if csv or not
        String checksum = AdminPortalUtil.generateChecksum(AdminPortalUtil.readFileBytes(file));
        bulkUploadValidation.validateFile(file, bulkUploadConfig, checksum);
        List<String> headers = readHeaders(bulkUploadConfig.getFileFormatConfig());
        // Upload file on s3
        String filePath = fileService.uploadFile(file);
        //insert file details in masterBulkUploadFile Table.
        MasterBulkUploadFile uploadFile = masterBulkUploadService.saveBulkUploadFile(masterCode, file, filePath, bulkUploadConfig.getId(), checksum);
        try {
            List<Map<String, String>> records = masterBulkUploadService.readRecords(file, headers);
            processUploadRecords(masterCode, masterService, uploadFile, records);
            logger.info("Bulk upload finished successfully for masterCode: {}, uploadFileId: {}", masterCode, uploadFile.getId());
            return buildUploadResponse(uploadFile);
        } catch (AdminPortalException  ex) {
            logger.error("Bulk upload failed for masterCode: {}, uploadFileId: {}. Reason: {}", masterCode, uploadFile.getId(), ex.getMessage());
            markUploadAsFailed(uploadFile, ex.getMessage());
            throw ex;
        }
    }

    private List<String> readHeaders(String fileFormatConfig) {
        try {
            return objectMapper.readValue(fileFormatConfig, new TypeReference<List<String>>() {});
        } catch (Exception exception) {
            throw new AdminPortalException(FAILED_ERROR_CODE, "Invalid bulk upload header configuration.");
        }
    }

    /**
     * Marks the given upload-file record as FAILED and persists a truncated remark (prefixed with the current correlation id) explaining the failure.
     *
     * @param uploadFile the upload-file record to mark as failed
     * @param remark     the failure reason
     */
    private void markUploadAsFailed(MasterBulkUploadFile uploadFile, String remark) {
        uploadFile.setParsingStatus(BulkUploadParsingStatus.FAILED);
        uploadFile.setFileStatus(BulkUploadFileStatus.FAILED);
        String remarkMessage= MDC.get(CORRELATION_ID)+"||"+StringUtils.truncate(remark,800);
        uploadFile.setRemark(StringUtils.defaultIfBlank(remarkMessage, "Bulk upload failed."));
        masterBulkUploadService.updateFileRecord(uploadFile);
        logger.info("Upload file id: {} marked as FAILED", uploadFile.getId());
    }

    private BaseMasterService<BaseMasterDto> resolveMasterService(String masterCode) {
        MasterType masterType = MasterType.getMasterType(masterCode);
        logger.info("Master request for: {}", masterType);
        return masterHandlerFactory.getMasterService(masterType.getCode());
    }

    /**
     * Processes all parsed bulk-upload records in batches: for each batch, applies the create/inactivate logic per row, tallies success/failure counts, persists
     * workflow entries and per-row detail records, then updates the overall upload file record with the final counts and status.
     *
     * @param masterCode   the master type code
     * @param masterService the resolved master service handler
     * @param uploadFile   the upload-file record being processed
     * @param records      the parsed CSV records
     */
    private void processUploadRecords(String masterCode, BaseMasterService<BaseMasterDto> masterService, MasterBulkUploadFile uploadFile, List<Map<String, String>> records) {
        long successCount = 0;
        long failedCount = 0;
        int batchSize = adminPortalConfig.getMasterBulkUploadBatchSize();
        logger.info("Processing {} record(s) for masterCode: {} in batches of {}", records.size(), masterCode, batchSize);
        for (int start = 0; start < records.size(); start += batchSize) {
            int end = Math.min(start + batchSize, records.size());
            List<Map<String, String>> batchRecords = records.subList(start, end);
            BulkUploadProcessResult processResult = processRecords(masterCode, masterService, batchRecords);
            List<BulkUploadRowResult> rowResults = processResult.rowResults();
            for (int index = 0; index < batchRecords.size(); index++) {
                BulkUploadRowResult rowResult = rowResults.get(index);
                if (BulkUploadRecordStatus.SUCCESS.name().equals(rowResult.getRecordStatus())) {
                    successCount++;
                } else {
                    failedCount++;
                }
            }
            if (processResult.workflowDtos() != null && !processResult.workflowDtos().isEmpty()) {
                masterWorkFlowService.save(processResult.workflowDtos());
            }
            masterBulkUploadService.saveMasterFileDtls(buildUploadFileDetails(uploadFile, batchRecords, rowResults));
        }
        updateMasterFile(uploadFile, records, successCount, failedCount);
        logger.info("Bulk upload completed for master code: {}. total: {}, success: {}, failed: {}", masterCode, records.size(), successCount, failedCount);
    }


    /**
     * Updates the upload-file record with final total/success/failed counts and sets its parsing/file status accordingly.
     *
     * @param uploadFile   the upload-file record to update
     * @param records      the full set of parsed records
     * @param successCount number of successfully processed records
     * @param failedCount  number of failed records
     */
    private void updateMasterFile(MasterBulkUploadFile uploadFile, List<Map<String, String>> records, long successCount, long failedCount) {
        uploadFile.setTotalRecords((long) records.size());
        uploadFile.setSuccessRecords(successCount);
        uploadFile.setFailedRecords(failedCount);
        uploadFile.setParsingStatus(BulkUploadParsingStatus.COMPLETED);
        uploadFile.setFileStatus(failedCount > 0 ? BulkUploadFileStatus.COMPLETED_WITH_ERRORS : BulkUploadFileStatus.COMPLETED);
        uploadFile.setRemark(String.format("Bulk upload processed. Success records: %d, Failed records: %d", successCount, failedCount));
        masterBulkUploadService.updateFileRecord(uploadFile);
        logger.info("Upload file id: {} updated with success: {}, failed: {}, status: {}", uploadFile.getId(), successCount, failedCount, uploadFile.getFileStatus());
    }

    private AdminPortalResponse<MasterBulkUploadResponse> buildUploadResponse(MasterBulkUploadFile uploadFile) {
        MasterBulkUploadResponse data = MasterBulkUploadResponse.builder().uploadFileId(uploadFile.getId()).masterType(uploadFile.getMasterType()).totalRecords(uploadFile.getTotalRecords()).successRecords(uploadFile.getSuccessRecords()).failedRecords(uploadFile.getFailedRecords()).parsingStatus(String.valueOf(uploadFile.getParsingStatus())).fileStatus(String.valueOf(uploadFile.getFileStatus())).remark(uploadFile.getRemark()).build();
        return AdminPortalResponse.<MasterBulkUploadResponse>builder().status(SUCCESS_RESPONSE_CODE).data(List.of(data)).total(1L).build();
    }

    /**
     * Processes a single batch of records for a given master handler, converting any unexpected {@link AdminPortalException} into a failure result for every
     * row in the batch rather than aborting the whole upload.
     *
     * @param masterType  the master type code
     * @param handler     the resolved master service handler
     * @param records     the batch of raw records
     * @return the batch's row results and any workflow entries to persist
     */
    private BulkUploadProcessResult processRecords(String masterType, BaseMasterService<BaseMasterDto> handler,  List<Map<String, String>> records) {
        try {
            List<BulkUploadRowResult> rowResults = new ArrayList<>(Collections.nCopies(records.size(), null));
            List<MasterWorkFlowDto> workflowDtos = new ArrayList<>();
            List<CreateMaster> createMasters = new ArrayList<>();
            List<CreateInactivation> createInactivation = new ArrayList<>();

            for (int index = 0; index < records.size(); index++) {
                Map<String, String> record = records.get(index);
                BaseMasterDto dto = toDto(record, handler.getDtoTargetClass());
                String status = record.get(STATUS);

                try {
                    if (StringUtils.isBlank(status) || ACTIVE.equalsIgnoreCase(status)) {
                        handler.validate(dto, WorkflowOperation.CREATE);
                        if (handler.findExisting(dto).isPresent()) {
                            rowResults.set(index, failureResult(RECORD_ALREADY_EXISTS));
                        } else {
                            createMasters.add(new CreateMaster(index, dto));
                        }
                        continue;
                    }

                    if (INACTIVE.equalsIgnoreCase(status)) {
                        Optional<BaseMasterDto> activeRecord = handler.findActiveApproved(dto);
                        if (activeRecord.isPresent()) {
                            createInactivation.add(new CreateInactivation(index, activeRecord.get()));
                        } else {
                            rowResults.set(index, failureResult(NO_VALID_RECORDS));
                        }
                        continue;
                    }

                    rowResults.set(index, failureResult("Unsupported status value: " + status));
                } catch (AdminPortalException ex) {
                    logger.info("Exception occurred while processing record: {}, error: {}", dto, ex.getMessage());
                    rowResults.set(index, failureResult(ex.getMessage()));
                }
            }

            createMaster(handler, createMasters, rowResults, workflowDtos);
            createInactivation(handler, createInactivation, rowResults, workflowDtos);
            return new BulkUploadProcessResult(rowResults, workflowDtos);
        } catch (AdminPortalException ex) {
            logger.error("Batch processing failed for masterType: {}. Reason: {}", masterType, ex.getMessage());
            return new BulkUploadProcessResult(records.stream().map(context -> failureResult(ex.getMessage())).toList(), List.of());
        }
    }

    private void createMaster(BaseMasterService<BaseMasterDto> handler, List<CreateMaster> createMasters, List<BulkUploadRowResult> rowResults, List<MasterWorkFlowDto> workflowDtos) {
        if (createMasters.isEmpty()) {
            return;
        }
        List<BaseMasterDto> dtos = createMasters.stream().map(CreateMaster::dto).toList();
        List<BaseMasterDto> savedDtos = handler.saveAll(dtos);
        for (int i = 0; i < createMasters.size(); i++) {
            CreateMaster createMaster = createMasters.get(i);
            BaseMasterDto savedDto = savedDtos.get(i);
            workflowDtos.add(handler.mapToMasterWorkFlowDto(savedDto, WorkflowOperation.CREATE, MessageFormat.format("Bulk upload {0} action performed.", WorkflowOperation.CREATE.name())));
            rowResults.set(createMaster.index(), BulkUploadRowResult.builder().recordStatus(BulkUploadRecordStatus.SUCCESS.name()).operation(WorkflowOperation.CREATE.name()).remark(CREATE_PENDING_REMARK).masterRecordId(savedDto.getId()).build());
        }
    }

    private void createInactivation(BaseMasterService<BaseMasterDto> handler, List<CreateInactivation> createInactivations, List<BulkUploadRowResult> rowResults, List<MasterWorkFlowDto> workflowDtos) {
        if (createInactivations.isEmpty()) {
            return;
        }
        handler.performAction(createInactivations.stream().map(createInactivation -> createInactivation.record().getId()).toList(), WorkflowOperation.INACTIVE);
        for (CreateInactivation createInactivation : createInactivations) {
            BaseMasterDto record = createInactivation.record();
            workflowDtos.add(handler.mapToMasterWorkFlowDto(record, WorkflowOperation.INACTIVE, MessageFormat.format("Bulk upload {0} action performed.", WorkflowOperation.INACTIVE.name())));
            rowResults.set(createInactivation.index(), BulkUploadRowResult.builder().recordStatus(BulkUploadRecordStatus.SUCCESS.name()).operation(WorkflowOperation.INACTIVE.name()).remark(INACTIVE_PENDING_REMARK).masterRecordId(record.getId()).build());
        }
    }

    private BulkUploadRowResult failureResult(String message) {
        String remarkMessage= MDC.get(CORRELATION_ID)+"||"+StringUtils.truncate(message,800);
        return BulkUploadRowResult.builder().recordStatus(BulkUploadRecordStatus.FAILED.name()).remark(remarkMessage).build();
    }



    private List<MasterBulkUploadFileDetail> buildUploadFileDetails(MasterBulkUploadFile uploadFile, List<Map<String, String>> records, List<BulkUploadRowResult> rowResults) {
        List<MasterBulkUploadFileDetail> fileDetails = new ArrayList<>(records.size());
        for (int index = 0; index < records.size(); index++) {
            fileDetails.add(mapDetail(uploadFile, records.get(index), rowResults.get(index)));
        }
        return fileDetails;
    }

    private MasterBulkUploadFileDetail mapDetail(MasterBulkUploadFile uploadFile, Map<String, String> record, BulkUploadRowResult rowResult) {
        MasterBulkUploadFileDetail detail = new MasterBulkUploadFileDetail();
        detail.setMasterUploadFileId(uploadFile.getId());
        detail.setRecordJson(toJson(record));
        detail.setStatus(rowResult.getRecordStatus());
        detail.setOperation(rowResult.getOperation());
        detail.setRemark(rowResult.getRemark());
        detail.setMasterRecordId(rowResult.getMasterRecordId());
        return detail;
    }

    private record CreateMaster(int index, BaseMasterDto dto) {
    }

    private record CreateInactivation(int index, BaseMasterDto record) {
    }

    private record BulkUploadProcessResult(List<BulkUploadRowResult> rowResults, List<MasterWorkFlowDto> workflowDtos) {
    }

    /**
     * This is to download the csv file based upon the status for the bulk upload data.
     * @param response HttpServletResponse
     * @param bulkDownloadRequest bulk download request object
     */
    public void downloadBulkMaster(HttpServletResponse response,BulkDownloadRequest bulkDownloadRequest) {
        bulkUploadValidation.validateDownloadBulkMaster(bulkDownloadRequest.getBulkId(),bulkDownloadRequest.getBulkStatus(), bulkDownloadRequest.getMasterCode());
        masterBulkUploadService.downloadBulkUploadMaster(response, bulkDownloadRequest.getBulkId(),bulkDownloadRequest.getBulkStatus(), bulkDownloadRequest.getMasterCode());
    }
}

===========================================================
    /**
     * Bulk uploads master data via a CSV file: validates and stores the file,
     * parses it into records, processes each record (create or inactivate,
     * depending on its status column), and persists per-row and overall
     * upload results.
     *
     * @param masterCode String
     * @param file       multiPartFile
     * @return MasterBulkUploadResponse
     */
    public AdminPortalResponse<MasterBulkUploadResponse> upload(String masterCode, MultipartFile file) {
        logger.info("Starting bulk upload for masterCode: {}, fileName: {}", masterCode, file.getOriginalFilename());
        BaseMasterService<BaseMasterDto> masterService = resolveMasterService(masterCode);
        MasterBulkUploadConfig bulkUploadConfig = masterBulkUploadService.getBulkUploadConfig(masterCode);
        // validate file if csv or not
        String checksum = AdminPortalUtil.generateChecksum(AdminPortalUtil.readFileBytes(file));
        bulkUploadValidation.validateFile(file, bulkUploadConfig, checksum);
        List<String> headers = readHeaders(bulkUploadConfig.getFileFormatConfig());
        // Upload file on s3
        String filePath = fileService.uploadFile(file);
        //insert file details in masterBulkUploadFile Table.
        MasterBulkUploadFile uploadFile = masterBulkUploadService.saveBulkUploadFile(masterCode, file, filePath, bulkUploadConfig.getId(), checksum);
        try {
            List<Map<String, String>> records = masterBulkUploadService.readRecords(file, headers);
            processUploadRecords(masterCode, masterService, uploadFile, records);
            logger.info("Bulk upload finished successfully for masterCode: {}, uploadFileId: {}", masterCode, uploadFile.getId());
            return buildUploadResponse(uploadFile);
        } catch (AdminPortalException  ex) {
            logger.error("Bulk upload failed for masterCode: {}, uploadFileId: {}. Reason: {}", masterCode, uploadFile.getId(), ex.getMessage());
            markUploadAsFailed(uploadFile, ex.getMessage());
            throw ex;
        }
    }

    private List<String> readHeaders(String fileFormatConfig) {
        try {
            return objectMapper.readValue(fileFormatConfig, new TypeReference<List<String>>() {});
        } catch (Exception exception) {
            throw new AdminPortalException(FAILED_ERROR_CODE, "Invalid bulk upload header configuration.");
        }
    }
============================================
package com.epay.admin.portal.service.master;

import com.epay.admin.portal.config.AdminPortalConfig;
import com.epay.admin.portal.dto.BaseMasterDto;
import com.epay.admin.portal.dto.master.MasterWorkFlowDto;
import com.epay.admin.portal.entity.admin.master.MasterBulkUploadConfig;
import com.epay.admin.portal.entity.admin.master.MasterBulkUploadFile;
import com.epay.admin.portal.entity.admin.master.MasterBulkUploadFileDetail;
import com.epay.admin.portal.exceptions.AdminPortalException;
import com.epay.admin.portal.externalservice.FileService;
import com.epay.admin.portal.model.request.master.*;
import com.epay.admin.portal.model.response.AdminPortalResponse;
import com.epay.admin.portal.model.response.master.MasterBulkUploadResponse;
import com.epay.admin.portal.util.AdminPortalUtil;
import com.epay.admin.portal.util.enums.*;
import com.epay.admin.portal.validator.MasterBulkUploadValidation;
import com.fasterxml.jackson.databind.JsonNode;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.sbi.epay.logging.utility.LoggerFactoryUtility;
import com.sbi.epay.logging.utility.LoggerUtility;
import jakarta.servlet.http.HttpServletResponse;
import lombok.RequiredArgsConstructor;
import org.apache.commons.lang3.StringUtils;
import org.slf4j.MDC;
import org.springframework.data.domain.Page;
import org.springframework.data.domain.Pageable;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;
import org.springframework.web.multipart.MultipartFile;

import java.text.MessageFormat;
import java.util.ArrayList;
import java.util.Collections;
import java.util.List;
import java.util.Map;
import java.util.Optional;

import static com.epay.admin.portal.util.AdminPortalConstants.*;
import static com.epay.admin.portal.util.AdminPortalUtil.*;

/**
 * Class Name: AdminMasterService
 * *
 * Author: V1018841(Saurabh Mahto)
 * <p>
 * Copyright (c) 2026 [State Bank of India]
 * All rights reserved
 * *
 * Version:1.0
 */
@Service
@RequiredArgsConstructor
public class AdminMasterService {
    private final LoggerUtility logger = LoggerFactoryUtility.getLogger(this.getClass());
    private final MasterHandlerFactory<BaseMasterDto> masterHandlerFactory;
    private final MasterWorkFlowService masterWorkFlowService;
    private final MasterBulkUploadService masterBulkUploadService;
    private final FileService fileService;
    private final AdminPortalConfig adminPortalConfig;
    private final MasterBulkUploadValidation bulkUploadValidation;
    private final ObjectMapper objectMapper;

    @Transactional
    public AdminPortalResponse<String> create(MasterRequest masterRequest, String masterCode) {
        BaseMasterService<BaseMasterDto> masterService = resolveMasterService(masterCode);

        BaseMasterDto masterDto = masterService.mapToMasterDto(masterRequest);
        logger.info("Create master request :{}", masterDto);

        masterService.validate(masterDto, WorkflowOperation.CREATE);
        logger.info("Request validated successfully.");

        masterDto = masterService.save(masterDto, WorkflowOperation.CREATE);
        logger.info("Record Created successfully for master code: {}", masterDto.getCode());

        MasterWorkFlowDto masterWorkFlowDto = masterService.mapToMasterWorkFlowDto(masterDto, WorkflowOperation.CREATE, MessageFormat.format(MASTER_ACTION_PERFORMED, WorkflowOperation.CREATE.name()));
        masterWorkFlowService.save(masterWorkFlowDto);
        logger.info("Master flow record updated for :{}", masterDto.getCode());

        return mapToResponse(MessageFormat.format(MASTER_ACTION_PERFORMED, WorkflowOperation.CREATE.name()));
    }

    @Transactional
    public AdminPortalResponse<String> update(MasterRequest masterRequest, String masterCode) {
        BaseMasterService<BaseMasterDto> masterService = resolveMasterService(masterCode);

        BaseMasterDto masterDto = masterService.mapToMasterDto(masterRequest);
        logger.info("Create master request :{}", masterDto);

        masterService.validate(masterDto, WorkflowOperation.UPDATE);
        masterDto = masterService.save(masterDto, WorkflowOperation.UPDATE);
        logger.info("Record update successfully for master code: {}", masterDto.getCode());

        MasterWorkFlowDto masterWorkFlowDto = masterService.mapToMasterWorkFlowDto(masterDto, WorkflowOperation.UPDATE, MessageFormat.format(MASTER_ACTION_PERFORMED, WorkflowOperation.UPDATE.name()));
        BaseMasterDto savedMasterDto = masterService.findByIds(List.of(masterDto.getId())).getFirst();
        masterWorkFlowService.update(masterWorkFlowDto, savedMasterDto, masterDto);
        logger.info("Master flow record updated for :{}", masterDto.getCode());

        return mapToResponse(MessageFormat.format(MASTER_ACTION_PERFORMED, WorkflowOperation.UPDATE.name()));
    }

    @Transactional(readOnly = true)
    public AdminPortalResponse<Map<String, Object>> search(String masterCode, MasterSearchRequest request, Pageable pageable) {
        BaseMasterService<BaseMasterDto> masterService = resolveMasterService(masterCode);
        Page<BaseMasterDto> page = masterService.search(request, pageable);
        return masterService.mapToResponse(page.getContent(), page.getTotalElements());
    }

    @Transactional
    public AdminPortalResponse<String> approve(MasterBulkRequest request, String masterCode) {
        return performAction(request, WorkflowOperation.APPROVE, List.of(MasterAction.PENDING_FOR_APPROVAL.name(), MasterAction.PENDING_FOR_INACTIVE.name(),  MasterAction.PENDING_FOR_REJECTED.name()), masterCode);
    }

    @Transactional
    public AdminPortalResponse<String> reject(MasterBulkRequest request, String masterCode) {
        return performAction(request, WorkflowOperation.REJECT, List.of(MasterAction.PENDING_FOR_APPROVAL.name(), MasterAction.PENDING_FOR_INACTIVE.name()), masterCode);
    }

    @Transactional
    public AdminPortalResponse<String> inactive(MasterBulkRequest request, String masterCode) {
        return performAction(request, WorkflowOperation.INACTIVE, List.of(MasterAction.APPROVED.name()), masterCode);
    }

    private AdminPortalResponse<String> performAction(MasterBulkRequest request, WorkflowOperation performOperation, List<String> allowedPreviousActions, String masterCode) {
        BaseMasterService<BaseMasterDto> masterService = resolveMasterService(masterCode);
        masterService.validateAction(request, performOperation.name());

        List<BaseMasterDto> masters = masterService.findByIds(request.getMasterIds());
        masterService.validateBulkMasters(masters, allowedPreviousActions, performOperation.name());
        logger.info("Request validated successfully for operation : {}.", performOperation.name());

        masterService.performAction(request.getMasterIds(), performOperation);
        logger.info(" {} : Operation performed successfully for master code: {}", performOperation.name(), masterCode);

        String remarks = StringUtils.isNotEmpty(request.getRemarks()) ? request.getRemarks() : MessageFormat.format(MASTER_ACTION_PERFORMED, performOperation.name());
        List<MasterWorkFlowDto> masterWorkFlowDtos = masters.stream().map(master -> masterService.mapToMasterWorkFlowDto(master, performOperation, remarks)).toList();
        masterWorkFlowService.save(masterWorkFlowDtos);
        logger.info("Master flow record saved of :{}, for Operation : {}", masterCode, performOperation.name());

        return mapToResponse(MessageFormat.format(MASTER_ACTION_PERFORMED, performOperation.name()));
    }


    private AdminPortalResponse<String> mapToResponse(String s) {
        return AdminPortalResponse.<String>builder().status(SUCCESS_RESPONSE_CODE).data(List.of(s)).count(1L).total(1L).build();
    }

    /**
     * Bulk uploads master data via a CSV file: validates and stores the file,
     * parses it into records, processes each record (create or inactivate,
     * depending on its status column), and persists per-row and overall
     * upload results.
     *
     * @param masterCode String
     * @param file       multiPartFile
     * @return MasterBulkUploadResponse
     */
    public AdminPortalResponse<MasterBulkUploadResponse> upload(String masterCode, MultipartFile file) {
        logger.info("Starting bulk upload for masterCode: {}, fileName: {}", masterCode, file.getOriginalFilename());
        BaseMasterService<BaseMasterDto> masterService = resolveMasterService(masterCode);
        MasterBulkUploadConfig bulkUploadConfig = masterBulkUploadService.getBulkUploadConfig(masterCode);
        // validate file if csv or not
        String checksum = AdminPortalUtil.generateChecksum(AdminPortalUtil.readFileBytes(file));
        bulkUploadValidation.validateFile(file, bulkUploadConfig, checksum);
        List<String> headers = extractHeaders(bulkUploadConfig.getFileFormatConfig());
        // Upload file on s3
        String filePath = fileService.uploadFile(file);
        //insert file details in masterBulkUploadFile Table.
        MasterBulkUploadFile uploadFile = masterBulkUploadService.saveBulkUploadFile(masterCode, file, filePath, bulkUploadConfig.getId(), checksum);
        try {
            List<Map<String, String>> records = masterBulkUploadService.readRecords(file, headers);
            processUploadRecords(masterCode, masterService, uploadFile, records);
            logger.info("Bulk upload finished successfully for masterCode: {}, uploadFileId: {}", masterCode, uploadFile.getId());
            return buildUploadResponse(uploadFile);
        } catch (AdminPortalException  ex) {
            logger.error("Bulk upload failed for masterCode: {}, uploadFileId: {}. Reason: {}", masterCode, uploadFile.getId(), ex.getMessage());
            markUploadAsFailed(uploadFile, ex.getMessage());
            throw ex;
        }
    }

    private List<String> extractHeaders(String fileFormatConfig) {
        try {
            JsonNode root = objectMapper.readTree(fileFormatConfig);
            JsonNode headersNode = root.get("headers");
            if (headersNode == null || !headersNode.isArray()) {
                throw new IllegalArgumentException("headers is missing or invalid");
            }
            List<String> headers = new ArrayList<>();
            headersNode.forEach(header -> headers.add(header.asText()));
            return headers;
        } catch (Exception exception) {
            throw new AdminPortalException(FAILED_ERROR_CODE, "Invalid bulk upload file format configuration.");
        }
    }

    /**
     * Marks the given upload-file record as FAILED and persists a truncated remark (prefixed with the current correlation id) explaining the failure.
     *
     * @param uploadFile the upload-file record to mark as failed
     * @param remark     the failure reason
     */
    private void markUploadAsFailed(MasterBulkUploadFile uploadFile, String remark) {
        uploadFile.setParsingStatus(BulkUploadParsingStatus.FAILED);
        uploadFile.setFileStatus(BulkUploadFileStatus.FAILED);
        String remarkMessage= MDC.get(CORRELATION_ID)+"||"+StringUtils.truncate(remark,800);
        uploadFile.setRemark(StringUtils.defaultIfBlank(remarkMessage, "Bulk upload failed."));
        masterBulkUploadService.updateFileRecord(uploadFile);
        logger.info("Upload file id: {} marked as FAILED", uploadFile.getId());
    }

    private BaseMasterService<BaseMasterDto> resolveMasterService(String masterCode) {
        MasterType masterType = MasterType.getMasterType(masterCode);
        logger.info("Master request for: {}", masterType);
        return masterHandlerFactory.getMasterService(masterType.getCode());
    }

    /**
     * Processes all parsed bulk-upload records in batches: for each batch, applies the create/inactivate logic per row, tallies success/failure counts, persists
     * workflow entries and per-row detail records, then updates the overall upload file record with the final counts and status.
     *
     * @param masterCode   the master type code
     * @param masterService the resolved master service handler
     * @param uploadFile   the upload-file record being processed
     * @param records      the parsed CSV records
     */
    private void processUploadRecords(String masterCode, BaseMasterService<BaseMasterDto> masterService, MasterBulkUploadFile uploadFile, List<Map<String, String>> records) {
        long successCount = 0;
        long failedCount = 0;
        int batchSize = adminPortalConfig.getMasterBulkUploadBatchSize();
        logger.info("Processing {} record(s) for masterCode: {} in batches of {}", records.size(), masterCode, batchSize);
        for (int start = 0; start < records.size(); start += batchSize) {
            int end = Math.min(start + batchSize, records.size());
            List<Map<String, String>> batchRecords = records.subList(start, end);
            BulkUploadProcessResult processResult = processRecords(masterCode, masterService, batchRecords);
            List<BulkUploadRowResult> rowResults = processResult.getRowResults();
            for (int index = 0; index < batchRecords.size(); index++) {
                BulkUploadRowResult rowResult = rowResults.get(index);
                if (BulkUploadRecordStatus.SUCCESS.name().equals(rowResult.getRecordStatus())) {
                    successCount++;
                } else {
                    failedCount++;
                }
            }
            if (processResult.getWorkflowDtos() != null && !processResult.getWorkflowDtos().isEmpty()) {
                masterWorkFlowService.save(processResult.getWorkflowDtos());
            }
            masterBulkUploadService.saveMasterFileDtls(buildUploadFileDetails(uploadFile, batchRecords, rowResults));
        }
        updateMasterFile(uploadFile, records, successCount, failedCount);
        logger.info("Bulk upload completed for master code: {}. total: {}, success: {}, failed: {}", masterCode, records.size(), successCount, failedCount);
    }


    /**
     * Updates the upload-file record with final total/success/failed counts and sets its parsing/file status accordingly.
     *
     * @param uploadFile   the upload-file record to update
     * @param records      the full set of parsed records
     * @param successCount number of successfully processed records
     * @param failedCount  number of failed records
     */
    private void updateMasterFile(MasterBulkUploadFile uploadFile, List<Map<String, String>> records, long successCount, long failedCount) {
        uploadFile.setTotalRecords((long) records.size());
        uploadFile.setSuccessRecords(successCount);
        uploadFile.setFailedRecords(failedCount);
        uploadFile.setParsingStatus(BulkUploadParsingStatus.COMPLETED);
        uploadFile.setFileStatus(failedCount > 0 ? BulkUploadFileStatus.COMPLETED_WITH_ERRORS : BulkUploadFileStatus.COMPLETED);
        uploadFile.setRemark(String.format("Bulk upload processed. Success records: %d, Failed records: %d", successCount, failedCount));
        masterBulkUploadService.updateFileRecord(uploadFile);
        logger.info("Upload file id: {} updated with success: {}, failed: {}, status: {}", uploadFile.getId(), successCount, failedCount, uploadFile.getFileStatus());
    }

    private AdminPortalResponse<MasterBulkUploadResponse> buildUploadResponse(MasterBulkUploadFile uploadFile) {
        MasterBulkUploadResponse data = MasterBulkUploadResponse.builder().uploadFileId(uploadFile.getId()).masterType(uploadFile.getMasterType()).totalRecords(uploadFile.getTotalRecords()).successRecords(uploadFile.getSuccessRecords()).failedRecords(uploadFile.getFailedRecords()).parsingStatus(String.valueOf(uploadFile.getParsingStatus())).fileStatus(String.valueOf(uploadFile.getFileStatus())).remark(uploadFile.getRemark()).build();
        return AdminPortalResponse.<MasterBulkUploadResponse>builder().status(SUCCESS_RESPONSE_CODE).data(List.of(data)).total(1L).build();
    }

    /**
     * Processes a single batch of records for a given master handler, converting any unexpected {@link AdminPortalException} into a failure result for every
     * row in the batch rather than aborting the whole upload.
     *
     * @param masterType  the master type code
     * @param handler     the resolved master service handler
     * @param records     the batch of raw records
     * @return the batch's row results and any workflow entries to persist
     */
    private BulkUploadProcessResult processRecords(String masterType, BaseMasterService<BaseMasterDto> handler,  List<Map<String, String>> records) {
        try {
            List<BulkUploadRowResult> rowResults = new ArrayList<>(Collections.nCopies(records.size(), null));
            List<MasterWorkFlowDto> workflowDtos = new ArrayList<>();
            List<CreateMaster> createMasters = new ArrayList<>();
            List<CreateInactivation> createInactivation = new ArrayList<>();

            for (int index = 0; index < records.size(); index++) {
                Map<String, String> record = records.get(index);
                BaseMasterDto dto = toDto(record, handler.getDtoTargetClass());
                String status = record.get(STATUS);

                try {
                    if (StringUtils.isBlank(status) || ACTIVE.equalsIgnoreCase(status)) {
                        handler.validate(dto, WorkflowOperation.CREATE);
                        if (handler.findExisting(dto).isPresent()) {
                            rowResults.set(index, failureResult(RECORD_ALREADY_EXISTS));
                        } else {
                            createMasters.add(new CreateMaster(index, dto));
                        }
                        continue;
                    }

                    if (INACTIVE.equalsIgnoreCase(status)) {
                        Optional<BaseMasterDto> activeRecord = handler.findActiveApproved(dto);
                        if (activeRecord.isPresent()) {
                            createInactivation.add(new CreateInactivation(index, activeRecord.get()));
                        } else {
                            rowResults.set(index, failureResult(NO_VALID_RECORDS));
                        }
                        continue;
                    }

                    rowResults.set(index, failureResult("Unsupported status value: " + status));
                } catch (AdminPortalException ex) {
                    logger.info("Exception occurred while processing record: {}, error: {}", dto, ex.getMessage());
                    rowResults.set(index, failureResult(ex.getMessage()));
                }
            }

            createMaster(handler, createMasters, rowResults, workflowDtos);
            createInactivation(handler, createInactivation, rowResults, workflowDtos);
            return BulkUploadProcessResult.builder().rowResults(rowResults).workflowDtos(workflowDtos).build();
        } catch (AdminPortalException ex) {
            logger.error("Batch processing failed for masterType: {}. Reason: {}", masterType, ex.getMessage());
            return BulkUploadProcessResult.builder().rowResults(records.stream().map(context ->
                    failureResult(ex.getMessage())).toList()).workflowDtos(List.of()).build();
        }
    }

    private void createMaster(BaseMasterService<BaseMasterDto> handler, List<CreateMaster> createMasters, List<BulkUploadRowResult> rowResults, List<MasterWorkFlowDto> workflowDtos) {
        if (createMasters.isEmpty()) {
            return;
        }
        List<BaseMasterDto> dtos = createMasters.stream().map(CreateMaster::dto).toList();
        List<BaseMasterDto> savedDtos = handler.saveAll(dtos);
        for (int i = 0; i < createMasters.size(); i++) {
            CreateMaster createMaster = createMasters.get(i);
            BaseMasterDto savedDto = savedDtos.get(i);
            workflowDtos.add(handler.mapToMasterWorkFlowDto(savedDto, WorkflowOperation.CREATE, MessageFormat.format("Bulk upload {0} action performed.", WorkflowOperation.CREATE.name())));
            rowResults.set(createMaster.index(), BulkUploadRowResult.builder().recordStatus(BulkUploadRecordStatus.SUCCESS.name()).operation(WorkflowOperation.CREATE.name()).remark(CREATE_PENDING_REMARK).masterRecordId(savedDto.getId()).build());
        }
    }

    private void createInactivation(BaseMasterService<BaseMasterDto> handler, List<CreateInactivation> createInactivations, List<BulkUploadRowResult> rowResults, List<MasterWorkFlowDto> workflowDtos) {
        if (createInactivations.isEmpty()) {
            return;
        }
        handler.performAction(createInactivations.stream().map(createInactivation -> createInactivation.record().getId()).toList(), WorkflowOperation.INACTIVE);
        for (CreateInactivation createInactivation : createInactivations) {
            BaseMasterDto record = createInactivation.record();
            workflowDtos.add(handler.mapToMasterWorkFlowDto(record, WorkflowOperation.INACTIVE, MessageFormat.format("Bulk upload {0} action performed.", WorkflowOperation.INACTIVE.name())));
            rowResults.set(createInactivation.index(), BulkUploadRowResult.builder().recordStatus(BulkUploadRecordStatus.SUCCESS.name()).operation(WorkflowOperation.INACTIVE.name()).remark(INACTIVE_PENDING_REMARK).masterRecordId(record.getId()).build());
        }
    }

    private BulkUploadRowResult failureResult(String message) {
        String remarkMessage= MDC.get(CORRELATION_ID)+"||"+StringUtils.truncate(message,800);
        return BulkUploadRowResult.builder().recordStatus(BulkUploadRecordStatus.FAILED.name()).remark(remarkMessage).build();
    }



    private List<MasterBulkUploadFileDetail> buildUploadFileDetails(MasterBulkUploadFile uploadFile, List<Map<String, String>> records, List<BulkUploadRowResult> rowResults) {
        List<MasterBulkUploadFileDetail> fileDetails = new ArrayList<>(records.size());
        for (int index = 0; index < records.size(); index++) {
            fileDetails.add(mapDetail(uploadFile, records.get(index), rowResults.get(index)));
        }
        return fileDetails;
    }

    private MasterBulkUploadFileDetail mapDetail(MasterBulkUploadFile uploadFile, Map<String, String> record, BulkUploadRowResult rowResult) {
        MasterBulkUploadFileDetail detail = new MasterBulkUploadFileDetail();
        detail.setMasterUploadFileId(uploadFile.getId());
        detail.setRecordJson(toJson(record));
        detail.setStatus(rowResult.getRecordStatus());
        detail.setOperation(rowResult.getOperation());
        detail.setRemark(rowResult.getRemark());
        detail.setMasterRecordId(rowResult.getMasterRecordId());
        return detail;
    }

    private record CreateMaster(int index, BaseMasterDto dto) {
    }

    private record CreateInactivation(int index, BaseMasterDto record) {
    }

    /**
     * This is to download the csv file based upon the status for the bulk upload data.
     * @param response HttpServletResponse
     * @param bulkDownloadRequest bulk download request object
     */
    public void downloadBulkMaster(HttpServletResponse response,BulkDownloadRequest bulkDownloadRequest) {
        bulkUploadValidation.validateDownloadBulkMaster(bulkDownloadRequest.getBulkId(),bulkDownloadRequest.getBulkStatus(), bulkDownloadRequest.getMasterCode());
        masterBulkUploadService.downloadBulkUploadMaster(response, bulkDownloadRequest.getBulkId(),bulkDownloadRequest.getBulkStatus(), bulkDownloadRequest.getMasterCode());
    }
}
