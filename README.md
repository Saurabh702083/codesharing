--liquibase formatted sql
--changeset Hariom:131_1
ALTER TABLE MIF_DOCUMENT_INFO ADD DEVIATION_STATUS VARCHAR2(50);
ALTER TABLE MIF_DOCUMENT_INFO_AUDIT ADD DEVIATION_STATUS VARCHAR2(50);
--------------
mif_consent_record			
id	UUID		
mobile_number	int		
mif_id	String		
touch_point_id	String		
purpose_set_id	String		
consent_version_id	String		
status	String: Active, Inactive, Expired, Revoked		
timestamp	Timestamp		
ccms_consent_id	String, UUID received from CCMS Write API		
sync_status	String: SYNCED, PENDING, FAILED		
			
privacy_notice			
id	UUID		
version	varchar		
touchpoint_id	varchar		
notice_text	text		
created_at	timestamp		
is_active	boolean		
			
purpose_master			
id	UUID		
purpose_code	varchar		
version	varchar		
title	varchar		
description	text		
is_essential	boolean		
validity_days	int		
touchpoint_id	varchar		
purpose_set_id	varchar		
			
			
