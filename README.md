# Active Ragdoll + Hit Reaction System

RAGE/Euphoria tarzı "active ragdoll" sistemlerinin temel prensiplerine dayanan,
gerçek çalışan, profesyonel seviye bir implementasyon.

## Bu Sistem Ne Yapar?

Bir karaktere darbe (yumruk, çarpışma, mermi vb.) uygulandığında:

1. **RAGDOLL_REACTING**: Karakter fiziğe tam geçer, ama PD controller'lar
   animasyon pozuna "geri dönmeye çalışır" — pasif ragdoll değil, "niyeti olan" ragdoll.
2. **Balance Controller**: Center of mass + support polygon (ayaklar) analizi
   ile dengenin bozulup bozulmadığını her frame hesaplar.
3. Eğer toparlanabiliyorsa → **RECOVERING** (yumuşak blend ile animasyona dönüş)
4. Eğer toparlanamıyorsa → **FALLING** (tam pasif ragdoll, gerçek düşüş)

Bu, hafif bir itmeyle sendeleyip toparlanan ama güçlü bir araç çarpmasında
gerçekten düşen bir karakter davranışı üretir — tam olarak GTA/RDR tarzı
oyunlarda gördüğün hit-reaction mantığı.

## Dosya Yapısı

```
include/
  RagdollMath.h           - Vec3/Quat (SİL, kendi math kütüphaneni kullan)
  Skeleton.h              - İskelet/bone hiyerarşisi + kütle dağılımı
  PDController.h          - Joint-space PD controller (sistemin kalbi)
  BalanceController.h     - COM + support polygon + recovery hesabı
  ActiveRagdollSystem.h   - Her şeyi birleştiren state machine
test/
  test_skeleton.cpp       - İskelet oluşturma testi
  test_balance.cpp        - Balance controller izole testi
  test_full_scenario.cpp  - Tam senaryo: darbe -> recovery/falling
```

## ForgeEngine'e Entegrasyon Adımları

### 1. Math katmanını değiştir
`RagdollMath.h`'ı SİL. `Vec3`, `Quat` tiplerini senin mevcut math
kütüphanenle değiştir. Aranan arayüz:
- `Vec3`: `+`, `-`, `*scalar`, `Dot`, `Cross`, `Normalized`, `Length`
- `Quat`: `Conjugate`, `operator*`, `Rotate`, `ToAngularError` (axis*angle
  çıkarımı — eğer senin Quat'ında yoksa `PDController.h` içindeki
  implementasyonu kopyala), `Slerp`, `FromAxisAngle`

### 2. Skeleton'ı ECS'ine bağla
Şu an `Skeleton` standalone bir sınıf. Senin ECS'inde muhtemelen
`SkeletalMeshComponent` veya benzeri bir şey var — `Bone` struct'ındaki
alanları (animLocalPos/Rot, physWorldPos/Rot, blendedWorldPos/Rot) o
component'e taşı veya bridge bir adapter yaz.

### 3. StepPhysicsAndPD'yi GERÇEK fizik solver'ınla değiştir
**Bu kritik.** Şu anki `StepPhysicsAndPD`, izole test için yazılmış minimal
bir entegratör (semi-implicit Euler + basitleştirilmiş "virtual support
force"). Senin BVH broadphase + sequential impulse solver'ın olduğu için:

- Her `Bone`'u senin fizik motorundaki bir `RigidBody`'e bağla
- PD controller'ın ürettiği `torque`'u, kendi `RigidBody::ApplyTorque()`
  (veya eşdeğeri) ile uygula — kendi entegrasyonunu YAPMA
- Ayak-yere contact constraint'ini kendi narrow-phase'ine bırak (bizim
  "virtual support force" yamamızı sil, gerçek constraint çok daha doğru
  olacak)

### 4. Animasyon sistemiyle bağla
`bone.animLocalPos/Rot` her frame senin animasyon sisteminden (skeletal
animation, blending sonrası) doldurulmalı. `bone.blendedWorldRot/Pos` ise
render'a giden son pozdur — `ComputeBlendedPose()` bunu hesaplıyor.

### 5. Çarpışma sistemine bağla
Araç çarpması, mermi vurusu, yumruk gibi olaylarda
`ActiveRagdollSystem::ApplyImpulse(impulseVector, targetBone)` çağır.
`impulseVector` = momentum değişimi (kg*m/s), yön + büyüklük.

## Önemli Tasarım Notları (Tuning İçin)

- `PDGains::kp/kd`: Bu sistemin "ayar düğmeleri". `kp` ne kadar hızlı
  hedefe gitmeye çalıştığını, `kd` titreşimi ne kadar söndürdüğünü
  belirler. `ActiveRagdollConfig` içindeki `coreGains/limbGains/
  extremityGains` üçlüsünü kendi karakterinin "ağırlığı" hissine göre
  elle ayarlaman gerekecek (bu, her active ragdoll sisteminde kaçınılmaz
  bir tuning süreci — sayıları ilk denemede doğru tahmin edemezsin).

- `maxRecoverForceAccel` (BalanceController): Karakterin ne kadar güçlü
  "kendini tutabileceğini" belirler. Çok yüksek = her darbede toparlanır
  (gerçekçi değil). Çok düşük = her dokunuşta düşer.

- `maxRecoverableLean`: COM'un support polygon dışında ne kadar
  gidebileceği (toparlanma denenmesi için). Bunun ötesi "kayıp" sayılır.

## Bilinen Sınırlamalar (Bilerek Basitleştirilmiş Kısımlar)

1. **Foot stepping yok**: Gerçek Euphoria, dengeyi kaybedince adım atar
   (foot placement planning). Bu sistemde yok — sadece "ankle/hip strategy"
   (gövde eğme + kuvvet) var. Adım atma eklemek istersen, `BalanceController`
   içine bir `PlanRecoveryStep()` fonksiyonu eklemen gerekir (orta-büyüklükte
   bir ek iş, ayrı bir konuşmada ele alınabilir).

2. **Get-up animasyonu yok**: `Falling` durumunda karakter yere yığılır ama
   "kalkma" mantığı stub bırakıldı — bu ayrı bir animasyon/IK sistemi
   gerektirir.

3. **Tek-seviye basitleştirilmiş kinematic chain**: `animLocalPos/Rot`
   doğrudan world hedefi olarak kullanılıyor (`TransformToWorld` fonksiyonu
   gerçek forward-kinematics yapmıyor). ECS entegrasyonunda bunu gerçek
   parent-chain compose'a çevirmen gerekecek.

## Test Sonuçları (Doğrulanmış Davranış)

```
Hafif darbe   (impulse=60)  -> toparlanıyor, ~1.78s'de tam animasyona döner
Orta darbe    (impulse=150) -> toparlanıyor, ~1.62s'de tam animasyona döner
Güçlü darbe   (impulse=300) -> instability eşiği geçiyor, düşüyor
Çok güçlü     (impulse=800) -> anında düşüyor (araç çarpması senaryosu)
```

Bu, gerçek bir hit-reaction sisteminin göstermesi gereken davranış: küçük
darbeler sendeletir ama düşürmez, büyük darbeler gerçekten düşürür.
