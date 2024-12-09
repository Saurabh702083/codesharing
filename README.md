# codesharing

public class EncryptionService {
    private static final LoggerUtility log = LoggerFactoryUtility.getLogger(EncryptionService.class);

    /**
     * Encrypts the given SecretKey using the another SecretKey.
     *
     * @param key          the SecretKey by which we want use for encrypting the given key
     * @param keyToEncrypt SecretKey which we want to encrypt
     * @return the String of encrypted and encoded KEK
     */
    public static String encryptSecretKey(@NonNull String key, @NonNull String keyToEncrypt, @NonNull EncryptionDecryptionAlgo algorithm, @NonNull GCMIvLength gcmIvLength, @NonNull GCMTagLength gcmTagLength) throws EncryptionDecryptionException {
        try {
            log.debug("EncryptionService encryptSecretKey key : {} keyToEncrypt: {} ", key, keyToEncrypt);
            byte[] encryptedKeK = encryptValueBySecretKey(algorithm, gcmIvLength, gcmTagLength, decodedSecretKey(keyToEncrypt), decodedSecretKey(key).getEncoded());
            return Base64.getEncoder().encodeToString(encryptedKeK);
        } catch (NoSuchAlgorithmException | NoSuchPaddingException | InvalidKeyException |
                 InvalidAlgorithmParameterException | IllegalArgumentException | UnsupportedOperationException |
                 IllegalStateException | IllegalBlockSizeException | BadPaddingException e) {
            log.error("EncryptionService :: encrypt ", e);
            throw new EncryptionDecryptionException(EncryptionDecryptionConstants.GENERIC_ERROR_CODE, EncryptionDecryptionConstants.GENERIC_ERROR_MESSAGE);
        }
    }

    /**
     * Encrypts the given plain-text using the SecretKey.
     *
     * @param key          the SecretKey by which we want use for encrypting the given text
     * @param value        String which we want to encrypt
     * @param algorithm    EncryptionDecryptionAlgo algorithm
     * @param gcmIvLength  algorithm iv length
     * @param gcmTagLength algorithm tag length
     * @return the byte[] array of encrypted plain-text
     */
    public byte[] encryptValue(@NonNull SecretKey key, @NonNull String value, @NonNull EncryptionDecryptionAlgo algorithm, @NonNull GCMIvLength gcmIvLength, @NonNull GCMTagLength gcmTagLength) throws EncryptionDecryptionException {
        try {
            log.info("EncryptionService :: encrypt  key : {}, value : {}, algorithm : {}, gcmIvLength : {}, gcmTagLength : {}", key, value, algorithm, gcmIvLength, gcmTagLength);
            return encryptValueBySecretKey(algorithm, gcmIvLength, gcmTagLength, key, value.getBytes());
        } catch (NoSuchAlgorithmException | NoSuchPaddingException | InvalidKeyException |
                 InvalidAlgorithmParameterException | IllegalArgumentException | UnsupportedOperationException |
                 IllegalStateException | IllegalBlockSizeException | BadPaddingException e) {
            log.error("EncryptionService :: encrypt ", e);
            throw new EncryptionDecryptionException(EncryptionDecryptionConstants.GENERIC_ERROR_CODE, EncryptionDecryptionConstants.GENERIC_ERROR_MESSAGE);
        }
    }
