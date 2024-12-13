---
layout: post
title: Checking rows integrity by adding a signature column to tables
date: 2024-12-13 18:03 +0100
categories:
- Hibernate
- Security
- Spring boot
tags:
- hibernate
- jpa
- signature
- integrity
- security
- advanced
description: Sign a table row, so your app will detect any external changes.
---

## Introduction

In this tutorial I explain how to add extra checks for data integrity on the row level, by adding a signature column which will contain a signature for all the other row columns.

For that purpose I will use spring framework (spring -boot) and hibernate.

## GitHub repository

A full and working example is available on this [GitHub repository](https://github.com/Ibrahimhammani/row-signature)

## Use case

If you store some sensitive data you might want to ensure that no one else besides your application has changed it; or else you consider that your data is no longer valid.

Let's take this as a starting point for what's next.

Assuming that we have a ``Payment`` entity that having following fields:

* ``id`` a ``Long`` represents the id of the row
* ``amount`` a ``BigDecimal`` represents the amount paid by the customer
* ``date`` a ``ZonedDateTime`` for the payment date and time
* ``address`` a ``String`` for the invoicing address

To those fields we will add a signature field.

* ``signature`` a ``String`` that will hold the signature of my row.

Before saving the entity we will generate a signature for the previous fields and then store this signature in the signature field; and when we read this entity from the database we will check if the signature we have corresponds to the row data; if not the row is considered as invalid and we will throw an exception.

This give the following entity code:

```java
@Entity
@Table(name = "payment")
@Data
public class Payment implements Serializable {

    private static final long serialVersionUID = 1L;

    @Id
    @GeneratedValue(strategy = GenerationType.SEQUENCE, generator = "sequenceGenerator")
    @SequenceGenerator(name = "sequenceGenerator")
    @Column(name = "id")
    private Long id;

    @NotNull
    @Column(name = "amount", nullable = false,precision = 21, scale = 2)
    private BigDecimal amount;

    @NotNull
    @Column(name = "date", nullable = false)
    private ZonedDateTime date;

    @NotNull
    @Column(name = "address", nullable = false)
    private String address;

    @Column(name = "signature", nullable = false)
    private String signature;

}
```

## Signature

There are multiple ways to generate a signature for our row, we could use MD5 with salt or we could simply use encryption.

Each way has its pros and cons; for example MD5 is fast but less secure; encryption is slow and secure.

To have both performance and security we choose to use HMAC to generate signatures.

## HMAC

The hash-based message authentication code ([HMAC](https://en.wikipedia.org/wiki/HMAC)); is a way to check messages (our row data) integrity and authenticity; by using a cryptographic hash function in combination with a secret key; In this tutorial we will use ``SHA512`` as a function.

In the lines below you can see how to implement that using ``Java`` and ``Spring boot``

```java
@Configuration
public class DbSecurityConfig {

    private static final String HMAC_ALGORITHM = "HmacSHA512";

    @Bean
    public Mac signatureHMacEncoder(@Value("${application.security.entity.signature.key}") String key) 
                throws NoSuchAlgorithmException, InvalidKeyException {
        SecretKeySpec secretKeySpec = new SecretKeySpec(key.getBytes(StandardCharsets.UTF_8), HMAC_ALGORITHM);
        Mac hMac = Mac.getInstance(HMAC_ALGORITHM);
        hMac.init(secretKeySpec);
        return hMac;
    }
}
```

The implementation is pretty simple: we instantiate a ``SecretKeySpec`` using the spring boot property ``application.security.entity.signature.key`` value as a key; This property is created only for that purpose.

Once we have the ``SecretKeySpec`` we use it as a param to create a ``Mac`` instance  using ``HmacSHA512``.

## Automate signature generation and validation

Hibernate has some great event listeners that allow it to run some business logic before or after these events happen; for instance we will use the 3 following events for automating signature generation and validation.

* ``PreInsertEventListener`` to generate signature before inserting the rows into data base; this event will allow us to get a row ``ID`` before entity insert; see [my previous tutorial for more details](https://ibrahimhammani.blogspot.com/2022/10/manipulating-entity-id-before-it-gets.html).
* ``PreUpdateEventListener`` to update signature before entity update
* ``PostLoadEventListener`` to check whether the signature of the row is valid or not.

In this tutorial I will avoid explaining ``spring-boot/hibernate`` configuration for event listeners but don't panic, the [GitHub repository linked above](https://github.com/Ibrahimhammani/row-signature) contains a working example, and [my previous tutorial for more details](https://ibrahimhammani.blogspot.com/2022/10/manipulating-entity-id-before-it-gets.html) explains that.

Notice that I wrote an utility class called ``DbSecurityUtils`` to make signature generation easier and to format some data before encryption.

Formatting data before signature generation will allow us to have a working code regardless of environment specific configuration; Including ``OS``, ``Local``,``Java`` version or event external system like ``RDBMS``.

```java
public class DbSecurityUtils {

    private static final DateTimeFormatter DATE_TIME_FORMATTER = DateTimeFormatter.ofPattern("MM/dd/yyyy - HH:mm:ss Z");
    private static final DecimalFormat DECIMAL_FORMAT = new DecimalFormat();

    static {
        DECIMAL_FORMAT.setMaximumFractionDigits(2);
        DECIMAL_FORMAT.setMinimumFractionDigits(2);
        DECIMAL_FORMAT.setGroupingUsed(false);
    }

    public static String generatePlainSignature(Payment payment) {
        return payment.getId() +
                "|" +
                (payment.getAmount() != null ? DECIMAL_FORMAT.format(payment.getAmount()) : null) +
                "|" +
                (payment.getDate() != null ? DATE_TIME_FORMATTER.format(payment.getDate()) : null) +
                "|" +
                payment.getAddress();
    }

}
```

## PreInsertEventListener

``PreInsertEventListener`` and ``PreUpdateEventListener`` will contain the same logic; Getting a plain message from the significant fields of the entity, using them to generate an ``HMAC`` and then update the signature field.

```java
@Component
@AllArgsConstructor
public class SignaturePreInsertEntityListener implements PreInsertEventListener {

    private final Mac signatureHMacEncoder;


    @Override
    public boolean onPreInsert(PreInsertEvent event) {
        try {
            Payment payment = (Payment) event.getEntity();
            if (payment.getId() == null) {
                throw new NotSingableException("Entity id is null");
            }
            String signature = Base64
                    .getEncoder()
                    .encodeToString(signatureHMacEncoder.doFinal(DbSecurityUtils.generatePlainSignature(payment)
                    .getBytes(StandardCharsets.UTF_8)));
            event.getState()[ArrayUtils.indexOf(
                    event.getPersister().getEntityMetamodel().getPropertyNames(), "signature")] =
                    signature;
            payment.setSignature(signature);
        } catch (NotSingableException e) {
            throw new RuntimeException(e);
        }
        return false;
    }
}
```

## PreUpdateEventListener

Will contain exactly the same code as the previous listener but overrides ``onPreUpdate`` instead of ``onPreInsert``

```java
@Component
@AllArgsConstructor
public class SignaturePreUpdateEntityListener implements PreUpdateEventListener {

    private final Mac signatureHMacEncoder;


    @Override
    public boolean onPreUpdate(PreUpdateEvent event) {
        // same business logic as previous
    }
}
```

## PostLoadEventListener

This listener verifies that the signature still corresponds to the significant row's data; if not an exception will be thrown.

```java
@Component
@RequiredArgsConstructor
public class SignaturePostLoadEntityListener implements PostLoadEventListener {

    protected final Mac signatureHMacEncoder;


    @Override
    public void onPostLoad(PostLoadEvent event) {
        try {
            Payment payment = (Payment) event.getEntity();
            String currentSignature = payment.getSignature();
            if (StringUtils.isBlank(currentSignature)) {
                throw new NoSignatureException("Entity is not signed");
            }
            String calculatedSignature = Base64
                    .getEncoder()
                    .encodeToString(signatureHMacEncoder.doFinal(DbSecurityUtils.generatePlainSignature(payment)
                    .getBytes(StandardCharsets.UTF_8)));
            if (!calculatedSignature.equals(currentSignature)) {
                throw new InvalidSignatureException("Entity signature is not valid");
            }

        } catch (InvalidSignatureException | NoSignatureException e) {
            throw new RuntimeException(e);
        }
    }
}
```

## Testing

The following basic test will check whether the data including the signature is correctly saved and restored.

```java
@SpringBootTest(classes = { RowSignatureApplication.class })
class PaymentRepositoryTest {
    @Autowired
    PaymentRepository paymentRepository;
    @Autowired
    Mac signatureHMacEncoder;

    private final ZonedDateTime now= ZonedDateTime.now();
    private final String address= "Some address";
    private final BigDecimal amount= new BigDecimal("99.99");

    @Test
    public void insertTest(){
        Payment payment=new Payment();

        payment.setDate(now);
        payment.setAddress(address);
        payment.setAmount(amount);

        Payment savedPayment=paymentRepository.save(payment);

        Optional<Payment> readPayment=paymentRepository.findById(savedPayment.getId());
        assertTrue(readPayment.isPresent());
        assertTrue(StringUtils.isNotBlank(readPayment.get().getSignature()));
        String signature = Base64
                .getEncoder()
                .encodeToString(signatureHMacEncoder.doFinal(DbSecurityUtils.generatePlainSignature(readPayment.get())
                        .getBytes(StandardCharsets.UTF_8)));
        assertEquals(signature,readPayment.get().getSignature());
    }
}
```

## Conclusion

That should work fine for the most case; should you have any question or suggestion please let me know.
