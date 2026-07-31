### *Internal User

- **Subscriber Cash In** (Dashboard button)
    
- **Merchant Cash In** (Dashboard button)
    
- **Pending Cash Out Request** (Dashboard button / Transfer Menu)
    
- **Agent Deposit** (Under the "Deposit" side menu)
    
- _Other Capabilities:_ Can manage "External Agent", "Subscriber", and "Merchant" lists.
    

### Distributor Agent


- **Resale Agent Cash In** (Listed as "Agent Cash In" on the side menu, used to send float to Resale Agents)
    
- _Other Capabilities:_ Can manage and view "My RESALE AGENT'S".
    

### **Resale Agent
- **Agent Cash In** (To send float down to standard Agents)
    
- **Resale Agent Cash Out** (To send funds back up to their Distributor Agent, e.g., Brian Johnson)
    
- _Other Capabilities:_ Can manage and view "My AGENT'S".
    

### Agent 

_From screenshots 2 and 3:_

- **Subscriber Cash In** (Dashboard button, to put digital money into a customer's wallet)
    
- **Agent Cash Out** (To send funds back up to their parent Resale Agent, e.g., Rian Kemr)
    
- _Other Capabilities:_ Can manage and view "Subscriber" lists.
```java
public HashMap<TransferType, String> getCommissionDisbursementTransferTypes() {  
    HashMap<TransferType, String> transferStringMap = new HashMap<>();  
  
    // Agent <-> Subscriber  
    transferStringMap.put(TransferType.SUBSCRIBER_CASH_IN, "Subscriber Cash In");  
    transferStringMap.put(TransferType.SUBSCRIBER_CASH_OUT, "Subscriber Cash Out");  
    transferStringMap.put(TransferType.AGENT_TO_SUBSCRIBER, "Agent to Subscriber");  
  
    // Agent hierarchy transfers  
    transferStringMap.put(TransferType.AGENT_TO_RESALE_AGENT_TRANSFER, "Agent to Resale Agent Transfer");  
    transferStringMap.put(TransferType.RESALE_TO_AGENT_TRANSFER, "Resale to Agent Transfer");  
  
    transferStringMap.put(TransferType.DISTRIBUTOR_TO_RESALE_AGENT_TRANSFER, "Distributor to Resale Agent Transfer");  
    transferStringMap.put(TransferType.RESALE_TO_DISTRIBUTOR_AGENT_TRANSFER, "Resale to Distributor Agent Transfer");  
  
    // Internal <-> External agent  
    transferStringMap.put(TransferType.CUSTOMER_CARE_TO_AGENT, "Customer Care to Agent");  
    transferStringMap.put(TransferType.INTERNAL_AGENT_TO_EXTERNAL_AGENT, "Internal Agent to External Agent");  
  
    // Agent related remittance/banking  
    transferStringMap.put(TransferType.EXTERNAL_AGENT_BANK_TRANSFER, "External Agent Bank Transfer");  
    transferStringMap.put(TransferType.EXTERNAL_AGENT_SEND_REMITTANCE, "External Agent Send Remittance");  
    transferStringMap.put(TransferType.AMAL_EXPRESS_TO_EXTERNAL_AGENT, "Amal Express to External Agent");  
  
    // Merchant <-> Agent  
    transferStringMap.put(TransferType.MERCHANT_TO_EXTERNAL_AGENT_TRANSFER, "Merchant to External Agent Transfer");  
    transferStringMap.put(TransferType.MERCHANT_CASH_IN, "Merchant Cash In");  
  
    return transferStringMap;  
}
```

```java
//SUBSCRIBER CASH OUT / CASH IN  
if (profile.getTransferType() == TransferType.SUBSCRIBER_CASH_OUT  
        || profile.getTransferType() == TransferType.SUBSCRIBER_CASH_IN) {  
  
    // For CASH_OUT -> agent wallet is receiver  
    // For CASH_IN  -> agent wallet is sender    String agentWalletId =  
            profile.getTransferType() == TransferType.SUBSCRIBER_CASH_OUT  
                    ? originalTransfer.getToAccountAggregateId()  
                    : originalTransfer.getFromAccountAggregateId();  
  
    if (role == UserType.AGENT) {  
  
        targetWalletId = agentWalletId;  
  
    } else if (role == UserType.RESALE_AGENT) {  
  
        Wallet agentWallet =  
                walletQueryRepository.findByAggregateId(agentWalletId);  
  
        UserInfo agentInfo =  
                userService.getUserInfoFromId(agentWallet.getUserId());  
  
        UserInfo resaleAgentInfo =  
                userService.getUserInfoFromId(agentInfo.getParentAgentId());  
  
        targetWalletId =  
                walletQueryRepository.findOneByUserIdAndType(  
                        resaleAgentInfo.getUserId(),  
                        WalletType.AGENT  
                ).getId();  
  
    } else if (role == UserType.DISTRIBUTOR_AGENT) {  
  
        Wallet agentWallet =  
                walletQueryRepository.findByAggregateId(agentWalletId);  
  
        UserInfo agentInfo =  
                userService.getUserInfoFromId(agentWallet.getUserId());  
  
        UserInfo resaleAgentInfo =  
                userService.getUserInfoFromId(agentInfo.getParentAgentId());  
  
        UserInfo distributorAgentInfo =  
                userService.getUserInfoFromId(  
                        resaleAgentInfo.getParentAgentId()  
                );  
  
        targetWalletId =  
                walletQueryRepository.findOneByUserIdAndType(  
                        distributorAgentInfo.getUserId(),  
                        WalletType.AGENT  
                ).getId();  
    }  
  
}  
  
//AGENT <-> RESALE AGENT TRANSFERS  
else if (profile.getTransferType() == TransferType.AGENT_TO_RESALE_AGENT_TRANSFER  
        || profile.getTransferType() == TransferType.RESALE_TO_AGENT_TRANSFER) {  
  
    // In both cases, resale agent wallet should be resolved  
    String resaleAgentWalletId =  
            profile.getTransferType() == TransferType.AGENT_TO_RESALE_AGENT_TRANSFER  
                    ? originalTransfer.getToAccountAggregateId() // receiver = resale agent  
                    : originalTransfer.getFromAccountAggregateId(); // sender = resale agent  
  
    if (role == UserType.RESALE_AGENT) {  
  
        targetWalletId = resaleAgentWalletId;  
  
    } else if (role == UserType.DISTRIBUTOR_AGENT) {  
  
        Wallet resaleAgentWallet =  
                walletQueryRepository.findByAggregateId(resaleAgentWalletId);  
  
        UserInfo resaleAgentInfo =  
                userService.getUserInfoFromId(resaleAgentWallet.getUserId());  
  
        UserInfo distributorAgentInfo =  
                userService.getUserInfoFromId(  
                        resaleAgentInfo.getParentAgentId()  
                );  
  
        targetWalletId =  
                walletQueryRepository.findOneByUserIdAndType(  
                        distributorAgentInfo.getUserId(),  
                        WalletType.AGENT  
                ).getId();  
    }  
}  
  
  
// RESALE AGENT <-> DISTRIBUTOR AGENT TRANSFERS  
else if (profile.getTransferType() == TransferType.RESALE_TO_DISTRIBUTOR_AGENT_TRANSFER  
        || profile.getTransferType() == TransferType.DISTRIBUTOR_TO_RESALE_AGENT_TRANSFER) {  
  
    // In both cases,  distributor agent wallet should be resolved  
    String distributorWalletId =  
            profile.getTransferType() == TransferType.RESALE_TO_DISTRIBUTOR_AGENT_TRANSFER  
                    ? originalTransfer.getToAccountAggregateId() // receiver = distributor  
                    : originalTransfer.getFromAccountAggregateId(); // sender = distributor  
  
  
    if (role == UserType.DISTRIBUTOR_AGENT) {  
  
        targetWalletId = distributorWalletId;  
    }  
}
```

```java
package com.vivacom.mfs.application.service;  
  
import com.vivacom.mfs.common.Constants;  
import com.vivacom.mfs.common.dto.TransferEventDto;  
import com.vivacom.mfs.common.info.UserInfo;  
import com.vivacom.mfs.core.api.MfsUtils;  
import com.vivacom.mfs.core.api.account.WalletType;  
import com.vivacom.mfs.core.api.accountTransfer.TransferStatus;  
import com.vivacom.mfs.core.api.accountTransfer.TransferType;  
import com.vivacom.mfs.core.api.exception.MFSException;  
import com.vivacom.mfs.core.api.transaction.profile.ServiceChargeProfileStatus;  
import com.vivacom.mfs.core.api.user.UserType;  
import com.vivacom.mfs.transaction.profile.query.entity.CommissionDisbursement;  
import com.vivacom.mfs.transaction.profile.query.entity.CommissionDisbursementProfile;  
import com.vivacom.mfs.transaction.profile.query.repository.CommissionDisbursementProfileRepository;  
import com.vivacom.mfs.user.service.UserService;  
import com.vivacom.mfs.wallet.query.entity.AccountTransfer;  
import com.vivacom.mfs.wallet.query.entity.CommissionInfo;  
import com.vivacom.mfs.wallet.query.entity.Wallet;  
import com.vivacom.mfs.wallet.query.repository.AccountTransferQueryRepository;  
import com.vivacom.mfs.wallet.query.repository.CommissionInfoQueryRepository;  
import com.vivacom.mfs.wallet.query.repository.WalletQueryRepository;  
import lombok.extern.slf4j.Slf4j;  
import org.joda.money.BigMoney;  
import org.springframework.beans.factory.annotation.Autowired;  
import org.springframework.stereotype.Service;  
  
import java.util.Date;  
import java.util.List;  
  
@Service  
@Slf4j  
public class CommissionDisbursementProcessingService {  
  
    @Autowired  
    private CommissionInfoQueryRepository commissionInfoQueryRepository;  
  
    @Autowired  
    private CommissionDisbursementProfileRepository profileRepository;  
  
    @Autowired  
    private AccountTransferQueryRepository accountTransferQueryRepository;  
  
    @Autowired  
    private WalletQueryRepository walletQueryRepository;  
  
    @Autowired  
    private GPayAccountTransferService gPayAccountTransferService;  
  
    @Autowired  
    private MfsUtils utils;  
  
    @Autowired  
    private UserService userService;  
  
    public void processDisbursement(TransferType transferType) {  
        log.info("Starting commission disbursement for transferType {}", transferType);  
  
        // Find active profile  
        CommissionDisbursementProfile profile = profileRepository.findByTransferTypeAndStatus(transferType, ServiceChargeProfileStatus.ACTIVE);  
        if (profile == null) {  
            log.warn("No active CommissionDisbursementProfile found for transferType {}", transferType);  
            throw new MFSException("No active CommissionDisbursementProfile found for transferType:  " + transferType);  
        }  
  
        // Get unpaid commission info for AMT03  
        List<CommissionInfo> unpaidCommissions = commissionInfoQueryRepository.findByAccountAggregateIdAndTransferTypeAndPaidStatus(Constants.AMT03_WALLET_AGGREGATE_ID, transferType, false);  
        if (unpaidCommissions == null || unpaidCommissions.isEmpty()) {  
            log.info("No unpaid commissions found in AMT03 for transferType {}", transferType);  
            throw new MFSException("No unpaid commissions found in Commission wallet for " +  transferType + " disbursement");  
        }  
  
        for (CommissionInfo commission : unpaidCommissions) {  
            try {  
                processSingleCommission(commission, profile);  
            } catch (Exception e) {  
                log.error("Failed to process commission {}", commission.getId(), e);  
            }  
        }  
    }  
  
    private void processSingleCommission(CommissionInfo commission, CommissionDisbursementProfile profile) throws Exception {  
        AccountTransfer originalTransfer = accountTransferQueryRepository.findOneByTransferAggregateId(commission.getTransferId());  
        if (originalTransfer == null) {  
            originalTransfer = accountTransferQueryRepository.findOne(commission.getTransferId());  
            if (originalTransfer == null) {  
                log.warn("Original transfer {} not found for commission {}", commission.getTransferId(), commission.getId());  
                throw new MFSException("Original transfer not found for commission " + commission.getTransferId());  
            }  
        }  
  
//        String senderTypeStr = utils.getSenderType(profile.getTransferType());  
//        String receiverTypeStr = utils.getReceiverType(profile.getTransferType());  
  
        for (CommissionDisbursement slab : profile.getDisbursements()) {  
            UserType role = slab.getRole();  
            if (role == UserType.ADMIN) {  
                // Keep in AMT03, do nothing  
                continue;  
            }  
  
            String targetWalletId = null;  
//            if (role.name().equalsIgnoreCase(senderTypeStr) || role.name().equalsIgnoreCase(senderTypeStr.replace(" ", "_"))) {  
//                targetWalletId = originalTransfer.getFromAccountAggregateId();  
//            } else if (role.name().equalsIgnoreCase(receiverTypeStr) || role.name().equalsIgnoreCase(receiverTypeStr.replace(" ", "_"))) {  
//                targetWalletId = originalTransfer.getToAccountAggregateId();  
//            }  
  
  
            //SUBSCRIBER CASH OUT / CASH IN            if (profile.getTransferType() == TransferType.SUBSCRIBER_CASH_OUT  
                    || profile.getTransferType() == TransferType.SUBSCRIBER_CASH_IN) {  
  
                // For CASH_OUT -> agent wallet is receiver  
                // For CASH_IN  -> agent wallet is sender                String agentWalletId =  
                        profile.getTransferType() == TransferType.SUBSCRIBER_CASH_OUT  
                                ? originalTransfer.getToAccountAggregateId()  
                                : originalTransfer.getFromAccountAggregateId();  
  
                if (role == UserType.AGENT) {  
  
                    targetWalletId = agentWalletId;  
  
                } else if (role == UserType.RESALE_AGENT) {  
  
                    Wallet agentWallet =  
                            walletQueryRepository.findByAggregateId(agentWalletId);  
  
                    UserInfo agentInfo =  
                            userService.getUserInfoFromId(agentWallet.getUserId());  
  
                    UserInfo resaleAgentInfo =  
                            userService.getUserInfoFromId(agentInfo.getParentAgentId());  
  
                    targetWalletId =  
                            walletQueryRepository.findOneByUserIdAndType(  
                                    resaleAgentInfo.getUserId(),  
                                    WalletType.AGENT  
                            ).getId();  
  
                } else if (role == UserType.DISTRIBUTOR_AGENT) {  
  
                    Wallet agentWallet =  
                            walletQueryRepository.findByAggregateId(agentWalletId);  
  
                    UserInfo agentInfo =  
                            userService.getUserInfoFromId(agentWallet.getUserId());  
  
                    UserInfo resaleAgentInfo =  
                            userService.getUserInfoFromId(agentInfo.getParentAgentId());  
  
                    UserInfo distributorAgentInfo =  
                            userService.getUserInfoFromId(  
                                    resaleAgentInfo.getParentAgentId()  
                            );  
  
                    targetWalletId =  
                            walletQueryRepository.findOneByUserIdAndType(  
                                    distributorAgentInfo.getUserId(),  
                                    WalletType.AGENT  
                            ).getId();  
                }  
  
            }  
  
            //AGENT <-> RESALE AGENT TRANSFERS  
            else if (profile.getTransferType() == TransferType.AGENT_TO_RESALE_AGENT_TRANSFER  
                    || profile.getTransferType() == TransferType.RESALE_TO_AGENT_TRANSFER) {  
  
                // In both cases, resale agent wallet should be resolved  
                String resaleAgentWalletId =  
                        profile.getTransferType() == TransferType.AGENT_TO_RESALE_AGENT_TRANSFER  
                                ? originalTransfer.getToAccountAggregateId() // receiver = resale agent  
                                : originalTransfer.getFromAccountAggregateId(); // sender = resale agent  
  
                if (role == UserType.RESALE_AGENT) {  
  
                    targetWalletId = resaleAgentWalletId;  
  
                } else if (role == UserType.DISTRIBUTOR_AGENT) {  
  
                    Wallet resaleAgentWallet =  
                            walletQueryRepository.findByAggregateId(resaleAgentWalletId);  
  
                    UserInfo resaleAgentInfo =  
                            userService.getUserInfoFromId(resaleAgentWallet.getUserId());  
  
                    UserInfo distributorAgentInfo =  
                            userService.getUserInfoFromId(  
                                    resaleAgentInfo.getParentAgentId()  
                            );  
  
                    targetWalletId =  
                            walletQueryRepository.findOneByUserIdAndType(  
                                    distributorAgentInfo.getUserId(),  
                                    WalletType.AGENT  
                            ).getId();  
                }  
            }  
  
  
            // RESALE AGENT <-> DISTRIBUTOR AGENT TRANSFERS  
            else if (profile.getTransferType() == TransferType.RESALE_TO_DISTRIBUTOR_AGENT_TRANSFER  
                    || profile.getTransferType() == TransferType.DISTRIBUTOR_TO_RESALE_AGENT_TRANSFER) {  
  
                // In both cases,  distributor agent wallet should be resolved  
                String distributorWalletId =  
                        profile.getTransferType() == TransferType.RESALE_TO_DISTRIBUTOR_AGENT_TRANSFER  
                                ? originalTransfer.getToAccountAggregateId() // receiver = distributor  
                                : originalTransfer.getFromAccountAggregateId(); // sender = distributor  
  
  
                if (role == UserType.DISTRIBUTOR_AGENT) {  
  
                    targetWalletId = distributorWalletId;  
                }  
            }  
  
  
            if (targetWalletId == null) {  
                log.warn("Could not determine target wallet for role {} on transfer {}", role, originalTransfer.getId());  
                continue;  
            }  
  
            BigMoney payoutAmount = utils.calculatePercentage(commission.getAmount(), slab.getPercentage());  
            if (payoutAmount.isZero() || payoutAmount.isNegative()) {  
                continue;  
            }  
  
            Wallet targetWallet = walletQueryRepository.findByAggregateId(targetWalletId);  
            if (targetWallet == null) {  
                log.warn("Target wallet {} not found", targetWalletId);  
                continue;  
            }  
  
            TransferType disbursementType = determineDisbursementTransferType(role);  
  
            UserInfo userInfoFromId = userService.getUserInfoFromId(1);  
  
            TransferEventDto transferDto = TransferEventDto.builder()  
                    .fromAccountAggregateId(Constants.AMT03_WALLET_AGGREGATE_ID)  
                    .toAccountAggregateId(targetWalletId)  
                    .fromAccountId(Constants.AMT03_WALLET_AGGREGATE_ID)  
                    .toAccountId(targetWalletId)  
                    .amount(payoutAmount)  
                    .serviceCharge(utils.getMoney(0.0))  
                    .transferType(disbursementType)  
                    .transferInitietedByType(com.vivacom.mfs.core.api.user.TransferInitietedByType.ADMIN)  
                    .transferInitiedByAggregateId(userInfoFromId.getAggregateId())  
                    .transferInitiedById(1)  
                    .transferStatus(TransferStatus.STARTED)  
                    .createdAt(new Date())  
                    .build();  
  
            String newTransferId = utils.getAccountTransferId(transferDto.getTransferType());  
            transferDto.setTransferAggregateId(newTransferId);  
  
            // Execute transfer  
            gPayAccountTransferService.processTransaction(transferDto);  
  
        }  
  
  
  
        // Update commission status  
        commission.setPaidStatus(true);  
        commissionInfoQueryRepository.save(commission);  
    }  
  
    private TransferType determineDisbursementTransferType(UserType role) {  
        switch (role) {  
            case AGENT:  
                return TransferType.COMMISSION_DISBURSEMENT_TO_AGENT;  
            case RESALE_AGENT:  
                return TransferType.COMMISSION_DISBURSEMENT_TO_RESALE_AGENT;  
            case DISTRIBUTOR_AGENT:  
                return TransferType.COMMISSION_DISBURSEMENT_TO_DISTRIBUTOR_AGENT;  
            case INTERNAL_AGENT:  
                return TransferType.COMMISSION_DISBURSEMENT_TO_INTERNAL_AGENT;  
            default:  
                throw new IllegalArgumentException("Unsupported disbursement role: " + role);  
        }  
    }  
}
```