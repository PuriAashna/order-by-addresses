<?php
/**
 * This is an extended model of Customer Address Entity for SLIM APIs
 * @package customer-ext
 * @author Hari CL <hari.lakshmipathy@embitel.com>
 */
class CustomerExt_Model_AddressApi extends CustomerExt_Model_Address {
    
    private $logFileName = 'API/customer.log'; // log file for customer api
    private $saveAddressIssue = 'address_counversion_issue.log'; // log file for customer api
    private $defaultStore = 'bigbazaar';

    private $_addressChangeLimit = 6;
    /**
     * PKD-633
     * Get customer addresses by id as required by the API signatures
     * @author Hari CL <hari.lakshmipathy@embitel.com>
     * @param int $customerId
     * @return \Transfer_Customer_AddressCollection
     */
    public function getCustomerActiveAddressById ($customerId) {

        if (!empty($customerId)) {
            $errorResponse = [];
            // Check if customer exists
            $customerModelObj = new CustomerExt_Model_Customer();
            $customerRecord = $customerModelObj->findById($customerId);
            if ($customerRecord) {
                // Prepare query to fetch addresses
                $addressTable = new DbTable_Customer_AddressTable();
                $regionTable = new DbTable_Customer_Address_RegionTable();
                $countryTable = new DbTable_CountryTable();
                $email  = empty($customerRecord->getEmail()) ? '' : $customerRecord->getEmail();
                $addressFields = [
                    'customerId'     => DbTable_Customer_AddressRow::FK_CUSTOMER,
                    'firstName'     => DbTable_Customer_AddressRow::FIRST_NAME,
                    'lastName'     => DbTable_Customer_AddressRow::LAST_NAME,
                    'addressId'     => DbTable_Customer_AddressRow::ID_CUSTOMER_ADDRESS,
                    'tag'           => DbTable_Customer_AddressRow::TAG,
                    'address1'      => DbTable_Customer_AddressRow::ADDRESS1,
                    'address2'      => DbTable_Customer_AddressRow::ADDRESS2,
                    'address3'      => DbTable_Customer_AddressRow::ADDRESS3,
                    'locality'      => DbTable_Customer_AddressRow::LOCALITY,
                    'subLocality'   => DbTable_Customer_AddressRow::SUB_LOCALITY,
                    'landmark'      => DbTable_Customer_AddressRow::LANDMARK,
                    'city'          => DbTable_Customer_AddressRow::CITY,
                    'country'       => DbTable_Customer_AddressRow::FK_COUNTRY,
                    'countryId'     => 'countryTbl.'.DbTable_CountryRow::ID_COUNTRY,
                    'regionId'     => 'regionTbl.'.DbTable_Customer_Address_RegionRow::ID_CUSTOMER_ADDRESS_REGION,
                    'pincode'       => DbTable_Customer_AddressRow::POSTCODE,
                    'lat'           => DbTable_Customer_AddressRow::LATITUDE,
                    'lon'           => DbTable_Customer_AddressRow::LONGITUDE,
                ];
                $regionFields = [
                    'region' => DbTable_Customer_Address_RegionRow::NAME
                ];
                $countryFields = [
                    'country' => DbTable_CountryRow::NAME
                ];

                $select = $addressTable->select();
                $select
                        ->from(['addressTbl' => $addressTable->getName() , 'countryTbl' => $countryTable->getName()], $addressFields)
                        ->joinLeft(['regionTbl' => $regionTable->getName()], 'addressTbl.' . DbTable_Customer_AddressRow::FK_CUSTOMER_ADDRESS_REGION
                                . ' = regionTbl.'.DbTable_Customer_Address_RegionRow::ID_CUSTOMER_ADDRESS_REGION, $regionFields)
                        ->joinLeft(['countryTbl' => $countryTable->getName()], 'addressTbl.' . DbTable_Customer_AddressRow::FK_COUNTRY
                                . ' = countryTbl.' . DbTable_CountryRow::ID_COUNTRY, $countryFields)
                        ->where(DbTable_Customer_AddressRow::FK_CUSTOMER." = ?", $customerId)
                        ->where(DbTable_Customer_AddressRow::IS_DEFAULT_SHIPPING." = ?", 1)
                        ->where(DbTable_Customer_AddressRow::IS_DEFAULT_BILLING." = ?", 1)
                        ->setIntegrityCheck(false);
                
                $addressRowset = $addressTable->fetchAll($select)->toArray();
//                $addressRowset = $addressRowset->toArray()
                if(!empty($addressRowset[0])) {
                    $addressRowset[0]['email'] = $email;
                }
                return ['addresses' => $addressRowset];
            } else {
                $errorResponse = ['responseCode' => 418, 'responseMessage' => 'Invalid Customer Id'];
            }
        } else {
            $errorResponse = ['responseCode' => 418, 'responseMessage' => 'Invalid Customer Id'];
        } // end: if customerId exists
        return $errorResponse;
    } // end: function getCustomerActiveAddressById
 
    /**
     * PKD-633
     * Get customer addresses by id as required by the API signatures
     * @author Hari CL <hari.lakshmipathy@embitel.com>
     * @param int $customerId
     * @return \Transfer_Customer_AddressCollection
     */
    public function getAddressesByCustomerId ($customerId, $storeType = NULL, $addressId = null) {
        

        if (!empty($customerId)) {
            $errorResponse = [];
            // Check if customer exists
            $customerModelObj = new CustomerExt_Model_Customer();
            $customerRecord = $customerModelObj->findById($customerId);
            if ($customerRecord) {
                // Prepare query to fetch addresses
                $addressTable = new DbTable_Customer_AddressTable();
                $regionTable = new DbTable_Customer_Address_RegionTable();
                $countryTable = new DbTable_CountryTable();

                $addressFields = [
                    'addressId'     => DbTable_Customer_AddressRow::ID_CUSTOMER_ADDRESS,
                    'tag'           => DbTable_Customer_AddressRow::TAG,
                    'address1'      => DbTable_Customer_AddressRow::ADDRESS1,
                    'address2'      => DbTable_Customer_AddressRow::ADDRESS2,
                    'address3'      => DbTable_Customer_AddressRow::ADDRESS3,
                    'locality'      => DbTable_Customer_AddressRow::LOCALITY,
                    'subLocality'   => DbTable_Customer_AddressRow::SUB_LOCALITY,
                    'landmark'      => DbTable_Customer_AddressRow::LANDMARK,
                    'city'          => DbTable_Customer_AddressRow::CITY,
                    'pincode'       => DbTable_Customer_AddressRow::POSTCODE,
                    'storeType'     => DbTable_Customer_AddressRow::STORE_TYPE,
                    'lat'           => DbTable_Customer_AddressRow::LATITUDE,
                    'lon'           => DbTable_Customer_AddressRow::LONGITUDE,
                    'isDefaultBilling'  => DbTable_Customer_AddressRow::IS_DEFAULT_BILLING,
                    'isDefaultShipping' => DbTable_Customer_AddressRow::IS_DEFAULT_SHIPPING,
                    'apartmentId' => DbTable_Customer_AddressRow::APARTMENT_ID
                ];
                $regionFields = [
                    'region' => DbTable_Customer_Address_RegionRow::NAME
                ];
                $countryFields = [
                    'country' => DbTable_CountryRow::NAME
                ];

                $select = $addressTable->select();

                if (empty($addressId)) {
                    $select
                        ->from(['addressTbl' => $addressTable->getName()], $addressFields)
                        ->joinLeft(['regionTbl' => $regionTable->getName()], 'addressTbl.' . DbTable_Customer_AddressRow::FK_CUSTOMER_ADDRESS_REGION
                            . ' = regionTbl.' . DbTable_Customer_Address_RegionRow::ID_CUSTOMER_ADDRESS_REGION, $regionFields)
                        ->joinLeft(['countryTbl' => $countryTable->getName()], 'addressTbl.' . DbTable_Customer_AddressRow::FK_COUNTRY
                            . ' = countryTbl.' . DbTable_CountryRow::ID_COUNTRY, $countryFields)
                        ->where(DbTable_Customer_AddressRow::FK_CUSTOMER . " = ?", $customerId)
                        ->where(DbTable_Customer_AddressRow::STATUS . " = ?", '1')
                        ->where(DbTable_Customer_AddressRow::IS_COUNPON_ADDRESS." = 0")
                        ->setIntegrityCheck(false);
                        
                    $storeType = empty($storeType) ? $this->defaultStore : $storeType;
                    if ($storeType == $this->defaultStore) {
                        $select->where(DbTable_Customer_AddressRow::STORE_TYPE . " = ? OR "
                            . DbTable_Customer_AddressRow::STORE_TYPE . " IS NULL", $storeType);
                    } else {
                        $select->where(DbTable_Customer_AddressRow::STORE_TYPE . " = ?", $storeType);
                    }
                } else {
                    $select
                        ->from(['addressTbl' => $addressTable->getName()], $addressFields)
                        ->joinLeft(['regionTbl' => $regionTable->getName()], 'addressTbl.' . DbTable_Customer_AddressRow::FK_CUSTOMER_ADDRESS_REGION
                            . ' = regionTbl.' . DbTable_Customer_Address_RegionRow::ID_CUSTOMER_ADDRESS_REGION, $regionFields)
                        ->joinLeft(['countryTbl' => $countryTable->getName()], 'addressTbl.' . DbTable_Customer_AddressRow::FK_COUNTRY
                            . ' = countryTbl.' . DbTable_CountryRow::ID_COUNTRY, $countryFields)
                        ->where(DbTable_Customer_AddressRow::FK_CUSTOMER . " = ?", $customerId)
                        ->where(DbTable_Customer_AddressRow::STATUS . " = ?", '1')
                        ->where(DbTable_Customer_AddressRow::ID_CUSTOMER_ADDRESS . " = ?", $addressId)
                        ->where(DbTable_Customer_AddressRow::IS_COUNPON_ADDRESS." = 0")
                        ->setIntegrityCheck(false);
                }

                $select = $select->order("FIELD(tag, 'work', 'home', 'house') DESC");
                $addressRowset = $addressTable->fetchAll($select);
                /**
                 * @todo send address counter details
                 */
                return ['addresses' => $addressRowset->toArray()];
            } else {
                $errorResponse = ['responseCode' => 418, 'responseMessage' => 'Invalid Customer Id'];
            }
        } else {
            $errorResponse = ['responseCode' => 418, 'responseMessage' => 'Invalid Customer Id'];
        } // end: if customerId exists
        return $errorResponse;
    } // end: function getAddressesByCustomerId


    /**
     * PKD-6656
     * Get customer address details
     * @param array $data
     */
    public function getAddressesDetails ($data) {
        return $this->getAddressesByCustomerId((int) $data['customerId'], null ,(int) $data['addressId']);
    } // end: function getAddressesByDetails

    /**
     * Save customer address
     * Task - PKD-631
     * @param type $data
     * @return success response
     */
    public function saveCustomerAddress($data = [] , $obpCheck = false) {
        $errorResponse = [];
        $walkInDelivery = false;
        if(array_key_exists("walkInDelivery", $data)){
            $walkInDelivery = ((boolean)$data['walkInDelivery'] === true) ? true : false;
        }
        try {
            if((int) $data['postcode']) {  // Accept all pincodes of 6 digit
                // Area not restricted based on pincode
                $table = new DbTable_Customer_AddressTable();
                if (!empty($data['customerId'])) {
                    // get customer data
                    $customerModelObj = new CustomerExt_Model_Customer();
                    $customerRecord = $customerModelObj->findByIdApi($data['customerId']);
                     
                    // if customer exists
                    if ($customerRecord) {
                        $addressChangeLimit = $this->_getAddressChangeLimit();
                        $addressChangeLimit = !empty ($addressChangeLimit) ? $addressChangeLimit : 6;  // if not configured in BOB
                        
                        /**
                         * Address change limit starts from the first address change,
                         * hence address change is calculated from the second address record onwards
                         */
                        
                        $loyaltyService     = new CustomerExt_Service_Loyalty();
                        $loyaltyData        = $loyaltyService->getLoyaltyByCustomer($data['customerId']);

                        if ($customerRecord->getAddressChangeCounter() >= $addressChangeLimit  && !empty($loyaltyData) && !$walkInDelivery) {
                            $errorResponse = [
                                'responseCode' => 422, 
                                'responseMessage' => 'Customer is not allowed to change the address anymore', 
                                'addressChangesLeft' => 0
                            ];
                        } else {
                        
                            // get first_name and last_name
                            $addressData[DbTable_Customer_AddressRow::FIRST_NAME] = empty ($customerRecord->getFirstName()) ? '' : $customerRecord->getFirstName(); 
                            $addressData[DbTable_Customer_AddressRow::MIDDLE_NAME] = empty ($customerRecord->getMiddleName()) ? '' : $customerRecord->getMiddleName(); 
                            $addressData[DbTable_Customer_AddressRow::LAST_NAME] = empty ($customerRecord->getLastName()) ? '' : $customerRecord->getLastName(); 
                            // end: if last_name
                            $addressData[DbTable_Customer_AddressRow::FK_CUSTOMER] = $data['customerId'];
                            $addressData[DbTable_Customer_AddressRow::ID_CUSTOMER_ADDRESS] = isset($data['addressId']) ? (int) $data['addressId'] : 0 ;
                            $addressData[DbTable_Customer_AddressRow::PREFIX] = isset($data['prefix']) ?  $data['prefix'] : null;
                            $addressData[DbTable_Customer_AddressRow::ADDRESS1] = isset($data['address1']) ?  $data['address1'] : null;
                            $addressData[DbTable_Customer_AddressRow::ADDRESS2] = isset($data['address2']) ?  $data['address2'] : null;
                            $addressData[DbTable_Customer_AddressRow::ADDRESS3] = isset($data['address3']) ?  $data['address3'] : null;
                            $addressData[DbTable_Customer_AddressRow::POSTCODE] = (int) $data['postcode'];

                            // Getting the phone number from customer table to be added in customer_address table
                            $customerContactModelObj = new CustomerExt_Model_ContactNumber();
                            $customerContactRecord = $customerContactModelObj->getActiveCustomerContactByCustomerId($data['customerId']);

                            $addressData[DbTable_Customer_AddressRow::PHONE] = isset($data['phone']) ?  $data['phone'] : !empty($customerContactRecord) ? $customerContactRecord->getContactNumber() : null;
                            $addressData[DbTable_Customer_AddressRow::LATITUDE] = isset($data['latitude']) ? (float) $data['latitude'] : null;
                            $addressData[DbTable_Customer_AddressRow::LONGITUDE] = isset($data['longitude']) ?  (float) $data['longitude'] : null;
                            $addressData[DbTable_Customer_AddressRow::CITY] = isset($data['city']) ?  $data['city'] : null;
                            $addressData[DbTable_Customer_AddressRow::LOCALITY] = isset($data['locality']) ? $data['locality'] : null;
                            $addressData[DbTable_Customer_AddressRow::SUB_LOCALITY] = isset($data['subLocality']) ? $data['subLocality'] : null;
                            $addressData[DbTable_Customer_AddressRow::LANDMARK] = isset($data['landmark']) ?  $data['landmark'] : null;
                            $addressData[DbTable_Customer_AddressRow::INSTRUCTIONS] = isset($data['instruction']) ?  $data['instruction'] : null;
                            $addressData[DbTable_Customer_AddressRow::TAG] = isset($data['tag']) ?  $data['tag'] : null;
                            $addressData[DbTable_Customer_AddressRow::COMPANY] = isset($data['company']) ? $data['company'] : null;
                            
                            /**
                             * PKD-4205 Walk-In-Delivery
                             */                            
                            // prepare data types and mandatory fields
                            $defaultAddressExists = ($walkInDelivery) ? $this->_checkDefaultAddressById($data['customerId']) : false;
                            $addressData[DbTable_Customer_AddressRow::IS_DEFAULT_BILLING] = ($walkInDelivery && $defaultAddressExists) ? 0 : 1;
                            $addressData[DbTable_Customer_AddressRow::IS_DEFAULT_SHIPPING] = ($walkInDelivery && $defaultAddressExists) ? 0 : 1;
                            
                            $isCounponAddress = isset($data['is_counpon_address']) && $data['is_counpon_address'] ? 1 : 0;
                            $addressData[DbTable_Customer_AddressRow::IS_COUNPON_ADDRESS] = $isCounponAddress;

                            
                            $customerInputCountryCode = 0;
                            // get fk_country
                            if (!empty($data['country'])) {
                                $countryTable = new DbTable_CountryTable();
                                $select = $countryTable->select();
                                $select
                                        ->from([$countryTable->getName()], ['id_country'])
                                        ->where(DbTable_CountryRow::NAME." = ?", $data['country']);

                                $record = $countryTable->fetchRow($select);
                                if (!empty ($record)) {
                                    $addressData[DbTable_Customer_AddressRow::FK_COUNTRY] = $record->toArray()['id_country'];
                                    $customerInputCountryCode = (int) $record->toArray()['id_country'];
                                } else {
                                    return $errorResponse = ['responseCode' => 418, 'responseMessage' => 'Country does not exist in system'];
                                }
                            } // end: if country

                            // get fk_customer_address_region
                            if (!empty($data['region'])) {
                                $regionTable = new DbTable_Customer_Address_RegionTable();
                                $select = $regionTable->select();
                                $select
                                        ->from([$regionTable->getName()], [DbTable_Customer_Address_RegionRow::ID_CUSTOMER_ADDRESS_REGION, DbTable_Customer_Address_RegionRow::FK_COUNTRY])
                                        ->where(DbTable_Customer_Address_RegionRow::NAME." = ?", $data['region']);
                                $record = $regionTable->fetchRow($select);
                                if (!empty ($record)) {
                                    /* check region and coutry matched or not  Added By Jyoti Jakhmola <jyoti.jakhmola@embitel.com> */
                                    if($customerInputCountryCode != $record->toArray()['fk_country']) {
                                        return $errorResponse = ['Region does not match with the country', 'responseCode' => 418];
                                    }
                                    $addressData[DbTable_Customer_AddressRow::FK_CUSTOMER_ADDRESS_REGION] = $record->toArray()['id_customer_address_region'];
                                }
                                else {
                                    
                                   //Check with pincode which have same region with diff name
                                    $states=array();
                                    if (!empty($data['postcode'])) {
                                        $db = Zend_Db_Table_Abstract::getDefaultAdapter();
                                        $select = $db->select();
                                        $select->from('pincode', array('state'))
                                                ->where('pincode = ?', $data['postcode']);
                                        $result = $db->fetchAll($select);
                                        foreach ($result as $row) {
                                            $states[] = $row['state'];
                                        }
                                    }
                                    //If city/region found same process 
                                    if (!empty($states) AND in_array($data['region'],$states)) {
                                        $regionTable = new DbTable_Customer_Address_RegionTable();
                                        $select = $regionTable->select();
                                        $select
                                                ->from([$regionTable->getName()], [DbTable_Customer_Address_RegionRow::ID_CUSTOMER_ADDRESS_REGION, DbTable_Customer_Address_RegionRow::FK_COUNTRY])
                                                ->where(DbTable_Customer_Address_RegionRow::NAME . " IN(?)", $states);
                                        $record = $regionTable->fetchRow($select);
                                        if (!empty($record)) {
                                            /* check region and coutry matched or not  Added By Jyoti Jakhmola <jyoti.jakhmola@embitel.com> */
                                            if ($customerInputCountryCode != $record->toArray()['fk_country']) {
                                                return $errorResponse = ['Region does not match with the country', 'responseCode' => 418];
                                            }
                                            $addressData[DbTable_Customer_AddressRow::FK_CUSTOMER_ADDRESS_REGION] = $record->toArray()['id_customer_address_region'];
                                        } else {
                                            return $errorResponse = ['Region does not exist in system', 'responseCode' => 418];
                                        }
                                    } else {
                                        return $errorResponse = ['Region does not exist in system', 'responseCode' => 418];
                                    }
                                } 
                            }// end: if region

                            // If no error
                            if (empty ($errorResponse)) {
                                // Update address if addressId is set in request
                                $whereAddressId = [DbTable_Customer_AddressRow::ID_CUSTOMER_ADDRESS .' = ? ' =>
                                                    $addressData[DbTable_Customer_AddressRow::ID_CUSTOMER_ADDRESS]];
                                $whereForCustomerId = [DbTable_Customer_AddressRow::FK_CUSTOMER .' = ? ' =>
                                                        $addressData[DbTable_Customer_AddressRow::FK_CUSTOMER]];

                                // if address record exists, update that record
                                if($addressData[DbTable_Customer_AddressRow::ID_CUSTOMER_ADDRESS]) {
                                    $select = $table->select();
                                    $select->from([$table->getName()])
                                           ->where(DbTable_Customer_AddressRow::FK_CUSTOMER." = ?", (int) $data['customerId'])
                                           ->where(DbTable_Customer_AddressRow::ID_CUSTOMER_ADDRESS .' = ? ', (int) $data['addressId']);
                                    if($table->fetchAll($select)->toArray()) {
                                        $table->update([DbTable_Customer_AddressRow::IS_DEFAULT_BILLING => 0 , 
                                                DbTable_Customer_AddressRow::IS_DEFAULT_SHIPPING => 0 ] , $whereForCustomerId );
                                        $table->update($addressData, array_merge($whereAddressId,$whereForCustomerId));
                                    } else {
                                       return ['responseCode' => 412,'responseMessage' => "addressId not associated with the given customerId"];
                                    }   
                                } else {
                                        // insert new address record
                                        if(!$walkInDelivery){
                                            $table->update([DbTable_Customer_AddressRow::IS_DEFAULT_BILLING => 0 , 
                                                DbTable_Customer_AddressRow::IS_DEFAULT_SHIPPING => 0 ] , $whereForCustomerId );
                                        }
                                            
                                    $addressData[DbTable_Customer_AddressRow::CREATED_AT] = date("Y-m-d H:i:s");
                                    $result =  $table->insert($addressData,$whereAddressId);
                                } // end: if id_customer_address

                                /**
                                 * PKD-4205 Walk-In-Delivery
                                 */
                                if($walkInDelivery !== true){
                                    // increment address change counter
                                    if ($addressCounter = $customerModelObj->incrementCustomerAddressChangeCounter($data['customerId'])) {
                                        $addressChangesLeft = $addressChangeLimit - $addressCounter;
                                    }
                                }
                                
                                $totalAddressCount = $this->getTotalAddressCountForCustomer($data['customerId']);
                               //PKD-2117  Obp address change check
                                if(!$obpCheck){
                                    $this->updateTransferObjectAddress((int) $data['customerId']);
                                }
                                
                                if($walkInDelivery){
                                    return ['responseCode' => 200,
                                            'responseMessage' => "Address saved successfully"
                                        ];
                                }else{
                                    return ['responseCode' => 200,
                                            'responseMessage' => "Address saved successfully, set as default",
                                            'totalAddresses' => $totalAddressCount,
                                            'addressChangesAllowed' => (int)$addressChangeLimit,
                                            'addressChangesLeft' => $addressChangesLeft
                                        ];
                                    
                                }
                            } // end: if no error
                        }
                    } else {
                        $errorResponse = ['responseCode' => 418, 'responseMessage' => "Customer does not exist"];
                    } // end: if
                } else {
                    $errorResponse = ['responseCode' => 418, 'responseMessage' => 'Invalid Customer Id'];
                } // end: if
            } else {
                $errorResponse = ['responseCode' => 418, 'responseMessage' => "Invalid Pincode"];
            }
        } catch(Exception $ex) {
            $errorResponse = ['responseCode' => 500 , 'responseMessage' => $ex->getMessage()];
        }
        return $errorResponse;
    } // end: function saveCustomerAddress


    /**
     * Save / edit address for customers
     * Task - PKD-6488
     * @param array $data
     * @return array
     */
    public function saveCustomerAddressV2($data = [] , $obpCheck = false) {
        $errorResponse = [];
        $isDefaultAddressEdited = false;
        Bob_Log::debug('Request Data = ' . var_export($data, true), $this->saveAddressIssue);
        try {
            $customerId = (int) $data['customerId'];
            $storeType = isset($data['storeType']) ? $data['storeType'] : $this->defaultStore;
            $addressId = !empty($data['addressId']) ? (int) $data['addressId'] : 0;
            $isCounponAddress = isset($data['is_counpon_address']) && $data['is_counpon_address'] ? 1 : 0;

            if(empty($data['postcode']) || !filter_var($data['postcode'], FILTER_VALIDATE_INT) || strlen($data['postcode'])!=6) {  // Accept all pincodes of 6 digit
                $errorResponse = ['responseCode' => 418, 'responseMessage' => "Invalid Pincode"];
            }
            elseif(empty($data['country'])){
                $errorResponse = ['responseCode' => 418, 'responseMessage' => 'Invalid Country'];
            }
            elseif(empty($customerId)){
                $errorResponse = ['responseCode' => 418, 'responseMessage' => 'Customer id is required'];	
            }
            else {

                // Area not restricted based on pincode
                $table = new DbTable_Customer_AddressTable();
                // get customer data
                $customerModelObj = new CustomerExt_Model_Customer();
                $customerRecord = $customerModelObj->findByIdApi($customerId);

                // if customer exists
                if ($customerRecord) {

                    // get first_name and last_name
                    $addressData[DbTable_Customer_AddressRow::FIRST_NAME] = empty ($customerRecord->getFirstName()) ? '' : $customerRecord->getFirstName();
                    $addressData[DbTable_Customer_AddressRow::MIDDLE_NAME] = empty ($customerRecord->getMiddleName()) ? '' : $customerRecord->getMiddleName();
                    $addressData[DbTable_Customer_AddressRow::LAST_NAME] = empty ($customerRecord->getLastName()) ? '' : $customerRecord->getLastName();
                    // end: if last_name
                    $addressData[DbTable_Customer_AddressRow::FK_CUSTOMER] = $customerId;
                    $addressData[DbTable_Customer_AddressRow::PREFIX] = isset($data['prefix']) ? $data['prefix'] : null;
                    $addressData[DbTable_Customer_AddressRow::ADDRESS1] = isset($data['address1']) ? $data['address1'] : null;
                    $addressData[DbTable_Customer_AddressRow::ADDRESS2] = isset($data['address2']) ? $data['address2'] : null;
                    $addressData[DbTable_Customer_AddressRow::ADDRESS3] = isset($data['address3']) ? $data['address3'] : null;
                    $addressData[DbTable_Customer_AddressRow::POSTCODE] = (int)$data['postcode'];
                    $addressData[DbTable_Customer_AddressRow::STORE_TYPE] = $storeType;

                    // Getting the phone number from customer table to be added in customer_address table
                    $customerContactModelObj = new CustomerExt_Model_ContactNumber();
                    $customerContactRecord = $customerContactModelObj->getActiveCustomerContactByCustomerId($customerId);

                    $addressData[DbTable_Customer_AddressRow::PHONE] = isset($data['phone']) ? $data['phone'] : !empty($customerContactRecord) ? $customerContactRecord->getContactNumber() : null;
                    $addressData[DbTable_Customer_AddressRow::LATITUDE] = isset($data['latitude']) ? (float)$data['latitude'] : null;
                    $addressData[DbTable_Customer_AddressRow::LONGITUDE] = isset($data['longitude']) ? (float)$data['longitude'] : null;
                   // $addressData[DbTable_Customer_AddressRow::CITY] = isset($data['city']) ? $data['city'] : null;
                    $addressData[DbTable_Customer_AddressRow::LOCALITY] = isset($data['locality']) ? $data['locality'] : null;
                    $addressData[DbTable_Customer_AddressRow::SUB_LOCALITY] = isset($data['subLocality']) ? $data['subLocality'] : null;
                    $addressData[DbTable_Customer_AddressRow::LANDMARK] = isset($data['landmark']) ? $data['landmark'] : null;
                    $addressData[DbTable_Customer_AddressRow::INSTRUCTIONS] = isset($data['instruction']) ? $data['instruction'] : null;
                    $addressData[DbTable_Customer_AddressRow::TAG] = isset($data['tag']) ? strtolower($data['tag']) : null;
                    $addressData[DbTable_Customer_AddressRow::COMPANY] = isset($data['company']) ? $data['company'] : null;
                    $addressData[DbTable_Customer_AddressRow::IS_DEFAULT_BILLING] = 0;
                    $addressData[DbTable_Customer_AddressRow::IS_DEFAULT_SHIPPING] = 0;
                    $addressData[DbTable_Customer_AddressRow::IS_COUNPON_ADDRESS] = $isCounponAddress;

                   // getting city
                    if (isset($data['city'])) {
                        $addressData[DbTable_Customer_AddressRow::CITY] = $data['city'];
                    }else if(!empty($data['postcode'])){
                            $pincodeTable = new DbTable_PincodeTable();
                            $select = $pincodeTable->select();
                            $select->from([$pincodeTable->getName()], ['city'])
                                ->where(DbTable_PincodeRow::PINCODE . " = ?", $data['postcode'])
                                ->order('id_pincode DESC')->limit(1);

                            $record = $pincodeTable->fetchRow($select);
                            if (!empty ($record)) {
                                $addressData[DbTable_Customer_AddressRow::CITY] = $record->toArray()['city'];
//                                print_r($addressData[DbTable_Customer_AddressRow::CITY]);
                            } else {
                                return $errorResponse = ['responseCode' => 418, 'responseMessage' => 'City does not exist in system'];
                            }
                     }

                    //PKD-10542 - Save Apartment Id (optional field)
                    $addressData[DbTable_Customer_AddressRow::APARTMENT_ID] = isset($data['apartmentId']) ? $data['apartmentId'] : null;

                    $customerInputCountryCode = 0;
                    // get fk_country
                    if (!empty($data['country'])) {
                        $countryTable = new DbTable_CountryTable();
                        $select = $countryTable->select();
                        $select
                            ->from([$countryTable->getName()], ['id_country'])
                            ->where(DbTable_CountryRow::NAME . " = ?", $data['country']);

                        $record = $countryTable->fetchRow($select);
                        if (!empty ($record)) {
                            $addressData[DbTable_Customer_AddressRow::FK_COUNTRY] = $record->toArray()['id_country'];
                            $customerInputCountryCode = (int)$record->toArray()['id_country'];
                        } else {
                            Bob_Log::debug('Country does not exist in system', $this->saveAddressIssue);
                            return $errorResponse = ['responseCode' => 418, 'responseMessage' => 'Country does not exist in system'];
                        }
                    } // end: if country


                    // get fk_customer_address_region
                    if (!empty($data['region'])) {
                        $regionTable = new DbTable_Customer_Address_RegionTable();
                        $select = $regionTable->select();
                        $select
                        ->from([$regionTable->getName()], [DbTable_Customer_Address_RegionRow::ID_CUSTOMER_ADDRESS_REGION, DbTable_Customer_Address_RegionRow::FK_COUNTRY])
                        ->where(DbTable_Customer_Address_RegionRow::NAME . " = ?", $data['region']);
                        $record = $regionTable->fetchRow($select);
                        if (!empty($record)) {
                            /* check region and coutry matched or not  Added By Jyoti Jakhmola <jyoti.jakhmola@embitel.com> */
                            if ($customerInputCountryCode != $record->toArray()['fk_country']) {
                                Bob_Log::debug('Region does not match with the country = ' . $data['region'], $this->saveAddressIssue);
                                return $errorResponse = ['Region does not match with the country', 'responseCode' => 418];
                            }
                            $addressData[DbTable_Customer_AddressRow::FK_CUSTOMER_ADDRESS_REGION] = $record->toArray()['id_customer_address_region'];
                        } else {
                            //Check with pincode which have same region with diff name
                            $states = array();
                            if (!empty($data['postcode'])) {
                                $db = Zend_Db_Table_Abstract::getDefaultAdapter();
                                $select = $db->select();
                                $select->from('pincode', array('state'))
                                ->where('pincode = ?', $data['postcode']);
                                $result = $db->fetchAll($select);
                                foreach ($result as $row) {
                                    $states[] = $row['state'];
                                }
                            }
                            //If city/region found same process 
                            if (!empty($states) and in_array($data['region'], $states)) {
                                Bob_Log::debug('States = ' . var_export($states, true), $this->saveAddressIssue);
                                $regionTable = new DbTable_Customer_Address_RegionTable();
                                $select = $regionTable->select();
                                $select
                                    ->from([$regionTable->getName()], [DbTable_Customer_Address_RegionRow::ID_CUSTOMER_ADDRESS_REGION, DbTable_Customer_Address_RegionRow::FK_COUNTRY])
                                    ->where(DbTable_Customer_Address_RegionRow::NAME . " IN(?)", $states);
                                $record = $regionTable->fetchRow($select);
                                if (!empty($record)) {
                                    /* check region and coutry matched or not  Added By Jyoti Jakhmola <jyoti.jakhmola@embitel.com> */
                                    if ($customerInputCountryCode != $record->toArray()['fk_country']) {
                                        Bob_Log::debug('Region does not match with the country and states', $this->saveAddressIssue);
                                        return $errorResponse = ['Region does not match with the country', 'responseCode' => 418];
                                    }
                                    $addressData[DbTable_Customer_AddressRow::FK_CUSTOMER_ADDRESS_REGION] = $record->toArray()['id_customer_address_region'];
                                } else {
                                    Bob_Log::debug('Region not found from db table', $this->saveAddressIssue);
                                    return $errorResponse = ['Region does not exist in system', 'responseCode' => 418];
                                }
                            } else {
                                Bob_Log::debug('Region not found States conition', $this->saveAddressIssue);
                                return $errorResponse = ['Region does not exist in system', 'responseCode' => 418];
                            }
                        }
                    }


                    // If no error
                    if (empty ($errorResponse)) {


                        //check tag

                        Zend_Db_Table::getDefaultAdapter()->beginTransaction();

                        if(!empty($data['tag'])){
                            $select = $table->select();
                            $select->from([$table->getName()])
                                ->where(DbTable_Customer_AddressRow::FK_CUSTOMER . " = ?", $customerId)
                                ->where(DbTable_Customer_AddressRow::TAG . ' = ? ', $data['tag'])
                                ->where(DbTable_Customer_AddressRow::ID_CUSTOMER_ADDRESS . ' != ? ', $addressId)
                                ->where(DbTable_Customer_AddressRow::STATUS . ' = ? ', "1");

                            if ($table->fetchAll($select)->toArray()) {

                                $whereForCustomerId = [DbTable_Customer_AddressRow::FK_CUSTOMER . ' = ? ' => $customerId,
                                    DbTable_Customer_AddressRow::TAG . ' = ? ' => $data['tag']];

                                $table->update([DbTable_Customer_AddressRow::TAG => "others"], $whereForCustomerId);

                            }
                        }

                        //Edit Address

                        if(!empty($addressId)){
                            $dataDelete = array(
                                "customerId" => $customerId,
                                "addressId" => $addressId
                            );
                            $addressDetails = $this->getAddressesByCustomerId($customerId,$storeType,$addressId);

                            $resp = $this->deleteAddress($dataDelete);

                            if(!empty($resp) && $resp['responseCode']!=200){
                                return $resp;
                            }

                            if(!empty($addressDetails['addresses'])){
                                $isDefaultAddressEdited = $addressDetails['addresses'][0]['isDefaultShipping'] == 1 ? true : false;
                            }
                        }

                        if(empty($addressId) || ($addressId && $isDefaultAddressEdited)) {
                            if (empty($storeType)) {
                                $whereForCustomerId = array(
                                    DbTable_Customer_AddressRow::FK_CUSTOMER . ' = ? ' =>
                                $addressData[DbTable_Customer_AddressRow::FK_CUSTOMER],
                                    DbTable_Customer_AddressRow::STORE_TYPE . ' IS NULL'
                                );
                            } else {
                                $whereForCustomerId = array(
                                    DbTable_Customer_AddressRow::FK_CUSTOMER . ' = ? ' =>
                                $addressData[DbTable_Customer_AddressRow::FK_CUSTOMER],
                                    DbTable_Customer_AddressRow::STORE_TYPE . ' = ? ' => $storeType
                                );
                             }


                            // insert new address record
                            $table->update([DbTable_Customer_AddressRow::IS_DEFAULT_BILLING => 0,
                                DbTable_Customer_AddressRow::IS_DEFAULT_SHIPPING => 0], $whereForCustomerId);

                            $addressData[DbTable_Customer_AddressRow::IS_DEFAULT_BILLING] = 1;
                            $addressData[DbTable_Customer_AddressRow::IS_DEFAULT_SHIPPING] = 1;
                        }

                        $addressData[DbTable_Customer_AddressRow::CREATED_AT] = date("Y-m-d H:i:s");
                        $table->insert($addressData);


                        $totalAddressCount = $this->getTotalAddressCountForCustomer($customerId);

                        Zend_Db_Table::getDefaultAdapter()->commit();

                        if (!$obpCheck) {
                            $this->updateTransferObjectAddress((int) $customerId, $storeType);
                        }

                        return ['responseCode' => 200,
                            'responseMessage' => empty($addressId) ? "Address saved successfully, set as default" : "Address updated successfully",
                            'totalAddresses' => $totalAddressCount,
                        ];
                    } // end: if no error




                } else {
                    $errorResponse = ['responseCode' => 418, 'responseMessage' => "Customer does not exist"];
                } // end: if

            }

        } catch(Exception $ex) {
            Bob_Log::debug('Exception ' . $ex->getMessage(), $this->saveAddressIssue);
            $errorResponse = ['responseCode' => 500 , 'responseMessage' => $ex->getMessage()];
        }
        return $errorResponse;
    } // end: function saveCustomerAddress


    /**
     * Mark address as defult address for customers
     * Task - PKD-6490
     * @param array $data
     * @return array
     */
    public function saveCustomerDefaultAddress($data = []){
        $customerId = $data['customerId'];
        $addressId = $data['addressId'];
        $storeType = empty($data['storeType']) ? $this->defaultStore : $data['storeType'];

        if(empty($customerId)){
            $response = ['responseCode' => 418, 'responseMessage' => 'Customer id is required'];
        }
        else if(empty($addressId)){
            $response = ['responseCode' => 418, 'responseMessage' => 'Address id is required'];
        }
        else{

            $table = new DbTable_Customer_AddressTable();
            $customerModelObj = new CustomerExt_Model_Customer();
            $customerRecord = $customerModelObj->findByIdApi($customerId);
            if(!empty($customerRecord)){
                $select = $table->select();
                $select->from([$table->getName()])
                    ->where(DbTable_Customer_AddressRow::FK_CUSTOMER . ' = ? ', $customerId)
                    ->where(DbTable_Customer_AddressRow::ID_CUSTOMER_ADDRESS . ' = ? ', $addressId);

                if ($storeType == $this->defaultStore) {
                    $select->where(DbTable_Customer_AddressRow::STORE_TYPE . " = ? OR "
                        . DbTable_Customer_AddressRow::STORE_TYPE . " IS NULL", $storeType);

                    $udpateArrayCustomer = array(
                        DbTable_Customer_AddressRow::FK_CUSTOMER . ' = ? ' => $customerId,
                        DbTable_Customer_AddressRow::STORE_TYPE . " = ? OR "
                            . DbTable_Customer_AddressRow::STORE_TYPE . " IS NULL" => $storeType
                    );
                    $udpateArrayAddress = array(
                        DbTable_Customer_AddressRow::ID_CUSTOMER_ADDRESS . ' = ? ' => $addressId,
                        DbTable_Customer_AddressRow::STORE_TYPE . " = ? OR "
                            . DbTable_Customer_AddressRow::STORE_TYPE . " IS NULL" => $storeType
                    );
                } else {
                    $select->where(DbTable_Customer_AddressRow::STORE_TYPE . ' = ? ', $storeType);
                    $udpateArrayCustomer = array(
                        DbTable_Customer_AddressRow::FK_CUSTOMER . ' = ? ' => $customerId,
                        DbTable_Customer_AddressRow::STORE_TYPE . ' = ? ' => $storeType
                    );
                    $udpateArrayAddress = array(
                        DbTable_Customer_AddressRow::ID_CUSTOMER_ADDRESS . ' = ? ' => $addressId,
                        DbTable_Customer_AddressRow::STORE_TYPE . ' = ? ' => $storeType
                    );
                }

                if ($table->fetchAll($select)->toArray()) {

                    $sql1 = $table->update([
                        DbTable_Customer_AddressRow::IS_DEFAULT_BILLING => 0,
                        DbTable_Customer_AddressRow::IS_DEFAULT_SHIPPING => 0
                    ], $udpateArrayCustomer);

                    $sql2 = $table->update([
                        DbTable_Customer_AddressRow::IS_DEFAULT_BILLING => 1,
                        DbTable_Customer_AddressRow::IS_DEFAULT_SHIPPING => 1
                    ], $udpateArrayAddress);

                    if($sql1 || $sql2){
                        $response = ['responseCode' => 200, 'responseMessage' => "Address successfully marked as default"];
                    }
                    else{
                        $response = ['responseCode' => 418, 'responseMessage' => "Something went wrong. Please try again"];
                    }

                } else {
                    $response = ['responseCode' => 418, 'responseMessage' => "Address doesnot belong to this customer"];
                }
            }
            else{
                $response = ['responseCode' => 418, 'responseMessage' => "Customer does not exist"];
            }

        }


        return $response;
    }


    /**
     * Delete address for customers
     * Task - PKD-6489
     * @param array $data
     * @return array
     */
    public function deleteAddress($data = []){
        $customerId = $data['customerId'];
        $addressId = $data['addressId'];

        if(empty($customerId)){
            $response = ['responseCode' => 418, 'responseMessage' => 'Customer id is required'];
        }
        else if(empty($addressId)){
            $response = ['responseCode' => 418, 'responseMessage' => 'Address id is required'];
        }
        else{

            $table = new DbTable_Customer_AddressTable();
            $customerModelObj = new CustomerExt_Model_Customer();
            $customerRecord = $customerModelObj->findByIdApi($customerId);
            if(!empty($customerRecord)){
                $whereAddressId = [DbTable_Customer_AddressRow::ID_CUSTOMER_ADDRESS . ' = ? ' => $addressId];

                $select = $table->select();
                $select->from([$table->getName()])
                    ->where(DbTable_Customer_AddressRow::FK_CUSTOMER . " = ?", $customerId)
                    ->where(DbTable_Customer_AddressRow::ID_CUSTOMER_ADDRESS . ' = ? ', $addressId);

                if ($table->fetchAll($select)->toArray()) {

                    $sql = $table->update([DbTable_Customer_AddressRow::IS_DEFAULT_BILLING => 0,
                        DbTable_Customer_AddressRow::IS_DEFAULT_SHIPPING => 0, DbTable_Customer_AddressRow::STATUS => 0], $whereAddressId);
                    if($sql){
                        $response = ['responseCode' => 200, 'responseMessage' => "Address deleted successfully"];
                    }
                    else{
                        $response = ['responseCode' => 418, 'responseMessage' => "Something went wrong. Please try again"];
                    }

                } else {
                    $response = ['responseCode' => 418, 'responseMessage' => "Address doesnot belong to this customer"];
                }
            }
            else{
                $response = ['responseCode' => 418, 'responseMessage' => "Customer does not exist"];
            }

        }


        return $response;
    }

    /**
     * PKD-631
     * Get customer default shipping address by id as required by the API signatures
     * @author Hari CL <hari.lakshmipathy@embitel.com>
     * @param int $customerId
     * @return array customer address
     */
    public function getDefaultShippingAddressByCustomerId ($customerId, $storeType = null) {

        if (!empty($customerId)) {
            $errorResponse = [];
            // Check if customer exists
            $customerModelObj = new CustomerExt_Model_Customer();
            $customerRecord = $customerModelObj->findById($customerId);

            // If customer record exists
            if ($customerRecord) {
                // Get max address change limit from configuration
                $addressChangeLimit = $this->_getAddressChangeLimit();

                // Prepare query to fetch addresses
                $addressTable = new DbTable_Customer_AddressTable();
                $regionTable = new DbTable_Customer_Address_RegionTable();
                $countryTable = new DbTable_CountryTable();

                $addressFields = [
                    'addressId'     => DbTable_Customer_AddressRow::ID_CUSTOMER_ADDRESS,
                    'tag'           => DbTable_Customer_AddressRow::TAG,
                    'address1'      => DbTable_Customer_AddressRow::ADDRESS1,
                    'address2'      => DbTable_Customer_AddressRow::ADDRESS2,
                    'address3'      => DbTable_Customer_AddressRow::ADDRESS3,
                    'locality'      => DbTable_Customer_AddressRow::LOCALITY,
                    'subLocality'   => DbTable_Customer_AddressRow::SUB_LOCALITY,
                    'landmark'      => DbTable_Customer_AddressRow::LANDMARK,
                    'city'          => DbTable_Customer_AddressRow::CITY,
                    'countryId'     => 'countryTbl.'.DbTable_CountryRow::ID_COUNTRY,
                    'regionId'      => 'regionTbl.'.DbTable_Customer_Address_RegionRow::ID_CUSTOMER_ADDRESS_REGION,
                    'regionName'    => 'regionTbl.'.DbTable_Customer_Address_RegionRow::NAME,
                    'pincode'       => DbTable_Customer_AddressRow::POSTCODE,
                    'storeType'     => DbTable_Customer_AddressRow::STORE_TYPE,
                    'lat'           => DbTable_Customer_AddressRow::LATITUDE,
                    'lon'           => DbTable_Customer_AddressRow::LONGITUDE,
                    'prefix'           => DbTable_Customer_AddressRow::PREFIX,
                    'company'           => DbTable_Customer_AddressRow::COMPANY,
                    'isDefaultBilling'  => DbTable_Customer_AddressRow::IS_DEFAULT_BILLING,
                    'isDefaultShipping' => DbTable_Customer_AddressRow::IS_DEFAULT_SHIPPING,
                    'apartmentId' => DbTable_Customer_AddressRow::APARTMENT_ID
                ];
                $regionFields = [
                    'region' => DbTable_Customer_Address_RegionRow::NAME
                ];
                $countryFields = [
                    'country' => DbTable_CountryRow::NAME
                ];

                $select = $addressTable->select();
                $select
                        ->from(['addressTbl' => $addressTable->getName()], $addressFields)
                        ->joinLeft(['regionTbl' => $regionTable->getName()], 'addressTbl.' . DbTable_Customer_AddressRow::FK_CUSTOMER_ADDRESS_REGION
                                . ' = regionTbl.'.DbTable_Customer_Address_RegionRow::ID_CUSTOMER_ADDRESS_REGION, $regionFields)
                        ->joinLeft(['countryTbl' => $countryTable->getName()], 'addressTbl.' . DbTable_Customer_AddressRow::FK_COUNTRY
                                . ' = countryTbl.' . DbTable_CountryRow::ID_COUNTRY, $countryFields)
                        ->where(DbTable_Customer_AddressRow::FK_CUSTOMER." = ?", $customerId)
                        ->where(DbTable_Customer_AddressRow::IS_DEFAULT_SHIPPING." = 1")
                        ->where(DbTable_Customer_AddressRow::IS_COUNPON_ADDRESS." = 0")
                        ->setIntegrityCheck(false);

                $storeType = empty($storeType) ? $this->defaultStore : $storeType;
                if ($storeType == $this->defaultStore) {
                    $select->where(DbTable_Customer_AddressRow::STORE_TYPE . " = ? OR "
                    . DbTable_Customer_AddressRow::STORE_TYPE . " IS NULL", $storeType);
                } else {
                    $select->where(DbTable_Customer_AddressRow::STORE_TYPE . " = ?", $storeType);
                }

                $addressRowset = $addressTable->fetchRow($select);
                
                if (!empty ($addressRowset) ) {
                    $addressesChanged = (int) $customerRecord->getAddressChangeCounter();
                    if($addressesChanged >= $addressChangeLimit){
                        $addressesChanged = 0;
                    }  else {
                        $addressesChanged = $addressChangeLimit - $addressesChanged;
                    }
                    $totalAddressCount = $this->getTotalAddressCountForCustomer($customerId);
                    $addressArray = $addressRowset->toArray();
                    $addressArray['totalAddresses'] = $totalAddressCount;
                    $addressArray['addressChangesAllowed'] = $addressChangeLimit;
                    $addressArray['addressChangesLeft'] = $addressesChanged;
                    return ['defaultShippingAddress'    => $addressArray];
                } else {
                    $return = [];
//                    if (VERSION == 'v1') {
//                        $return["defaultShippingAddress"] = $this->_getEmptyAddressFields();
//                    }
                    return $return;
                }
            } else {
                $errorResponse = ['responseCode' => 418, 'responseMessage' => 'Invalid Customer Id'];
            }
        } else {
            $errorResponse = ['responseCode' => 418, 'responseMessage' => 'Invalid Customer Id'];
        } // end: if customerId exists
        return $errorResponse;
    } // end: function getAddressesByCustomerId
    
    
    public function updateTransferObjectAddress($customerId, $storeType = null) {
        $orderServiceObj = new OrderprocessingExt_Service_OrderprocessApi();
        $orderServiceObj->updateTransferObjectAddress($customerId, $storeType);
    }
    
    /**
     * @task : method to get the total no. of addresses for the customer
     * @author Jyoti Jakhmola <jyoti.jakhmola@embitel.com>
     * @param : integer $customerId  mandatory
     */
    public function getTotalAddressCountForCustomer($customerId){
        try {
            $table = new DbTable_Customer_AddressTable();
            $select = $table->select();
            $select->from([$table->getName()], ['count' => 'COUNT(*)'])
                   ->where(DbTable_Customer_AddressRow::FK_CUSTOMER." = ?", (int) $customerId)
                   ->where(DbTable_Customer_AddressRow::STATUS." = ?", "1");
            $record = $table->fetchAll($select);
            if (!empty($record)) {
                $record = $record->toArray();
                return isset($record[0]['count']) ? $record[0]['count'] : 0;
            } else {
                return 0;
            }
        } catch (Exception $ex) {
            $exceptionMsg = $ex->getMessage()." FILE : ".$ex->getFile()." LINE NO. : ".$ex->getLine();
            Bob_Log::debug(__FUNCTION__.' EXCEPTION : '.
                    var_export($exceptionMsg, true), $this->logFileName);
        }
    }
    
    /**
     * @task : PKD-4205 | method to check default address for the customer exists
     * @author : Thejas N <thejas.nagabusan@embitel.com>
     * @param : integer $customerId  mandatory
     */
    private function _checkDefaultAddressById($customerId) {
        try {
            $table = new DbTable_Customer_AddressTable();
            $select = $table->select();
            $select ->from([$table->getName()], ['count' => 'COUNT(*)'])
                    ->where(DbTable_Customer_AddressRow::FK_CUSTOMER." = ?", (int) $customerId)
                    ->where(DbTable_Customer_AddressRow::IS_DEFAULT_SHIPPING." = 1")
                    ->where(DbTable_Customer_AddressRow::IS_DEFAULT_BILLING." = 1");
            $record = $table->fetchAll($select);
            if (!empty($record)) {
                $record = $record->toArray();
                return isset($record[0]['count']) ? $record[0]['count'] : 0;
            } else {
                return 0;
            }
        } catch (Exception $ex) {
            $exceptionMsg = $ex->getMessage()." FILE : ".$ex->getFile()." LINE NO. : ".$ex->getLine();
            Bob_Log::debug(__FUNCTION__.' EXCEPTION : '.
                    var_export($exceptionMsg, true), $this->logFileName);
        }
    }
    
    private function _getEmptyAddressFields() {
        $addressChangeLimit = $this->_getAddressChangeLimit();
        return [
            "addressId" => "",
            "tag" => "",
            "address1" => "",
            "address2" => "",
            "address3" => "",
            "locality" => "",
            "subLocality" => "",
            "landmark" => "",
            "city" => "",
            "countryId" => "",
            "regionId" => "",
            "regionName" => "",
            "pincode" => "",
            "lat" => "",
            "lon" => "",
            "prefix" => "",
            "company" => "",
            "isDefaultBilling" => "",
            "isDefaultShipping" => "",
            "region" => "",
            "country" => "",
            "totalAddresses" => 0,
            "addressChangesAllowed" => $addressChangeLimit,
            "addressChangesLeft" => $addressChangeLimit
        ];
    }

    /**
     * Gets the address change limit for customers from CustomerExt configuration in BOB backend
     * @return int $addressChangeLimit
     */
    private function _getAddressChangeLimit()
    {
        $customerExtConfig  = new ConfigurationExt_Service_Loader_Namespace('customer-ext', true);
        $addressChangeLimit = (int) $customerExtConfig->getValue('CUSTOMER_ADDRESS_COUNTER');
        $addressChangeLimit = !empty($addressChangeLimit) ? $addressChangeLimit : $this->_addressChangeLimit;
        return $addressChangeLimit;
    }
    
} // end: class CustomerExt_Model_AddressApi
