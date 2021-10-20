---
title:  "Retrofit2, Hilt를 활용한 API 호출 사용기"
excerpt: "Retrofit2, Hilt, Coroutines, Flow 를 조합해서 만든 API 호출 기능 입니다."

categories:
  - Android
tags:
  - Android
  - Retrofit2
  - Hilt
  - Coroutines
  - Flow
last_modified_at: 2021-10-20T21:00:00+09:00
---
## 배경
2019년 개발 당시 네트워크 라이브러리들은 다양했습니다. 그 중 iOS의 Alamofire의 구조와 비슷한 네트워크 라이브러리를 찾았었습니다.  Builder 패턴으로 만들어진 [Fast Android Networking](https://github.com/amitshekhariitbhu/Fast-Android-Networking) 은 매우 매력적으로 다가왔습니다.
사용하기 간편했으며, okHttp 기반의 라이브러리이기 때문에 안정성 또한 좋다고 판단하여 해당 라이브러리로 네트워크를 구성하였습니다.

이후 시간이 지나 MVVM 및 Clean Architecture 등 다양한 Android 개발 방식들이 만들어지면서, 
[Fast Android Networking](https://github.com/amitshekhariitbhu/Fast-Android-Networking) 만으로는 개발 방식을 따라가기에 버겁다는 인식을 하였습니다.(참고로 해당 라이브러리로 MVVM 구조까지도 만들었지만, 완벽하진 않았습니다.)

또한 면접을 진행하면서, 테스트 코드들이 모두 Clean Architecture 방식으로 구현되어 있었으며, di 주입을 통한 코드의 간결화를 한 것을 보고 추후 협업에 있어서 현재의 구현 방법으로는 어려움이 있겠다는 판단을 하였습니다.

마지막으로 추후에 테스트 코드를 작성해서 코드 안정성을 위해서는 개선이 필요하다는 결론을 지었습니다.
  
### 정리
1. 기존 네트워크 라이브러리로는 새로운 개발 방식을 따라가기 어려움
2. 신규 입사자들과 협업을 위해선 동일한 코드 이해도가 필요
3. 테스트 코드로 코드 안정성 확보 필요

## 정의
![](/assets/2021-10-20-Retrofit-Hilt-Part1/clean_architecture.png)  
실제로 공부하면서 느낀 것은 같은 정보를 가지고 다양한 해석이 나오고 다양한 방식 및 현재 상황에 맞는 방식으로 구현한다는 것을 확인할 수 있었습니다.

그래서 제가 정의한 것은 위에 스크린샷과 같습니다.

User는 사용자이며, 행위를 했을 때 Presentation Layer에게 전달을 합니다. 그럼 그안에 있는 View의 이벤트를 통해 ViewModel에게 행위를 전달 합니다.

ViewModel에서는 UseCase만 참조하며, UseCase에서 실제 비지니스 로직을 구현 합니다. 이 때 Presentation Layer에 데이터를 전달하기 위한 Model 클래스를 구현해서 데이터를 전달 합니다.  마지막으로 Data Layer와 Domain Layer 끼리 소통하기 위한 Repository Interface 구조를 만들어 놓습니다.

마지막으로 Data Layer에서는 API 통신 혹은 Local DB 처리를 위한 Source 기능을 구현하고, Data Layer 안에서 사용될 Data Class인 Entity를 만들었습니다.  마지막으로 Domain Layer와 실제 소통을 위해 Repository Interface를 상속받는 클래스를 구현합니다.

## 패키지 구조
![](/assets/2021-10-20-Retrofit-Hilt-Part1/package_struct.png)  
위에서 설명한 것처럼 data, domain, presentation을 기본으로 나누고, di는 hilt를 활용해서 주입을 위해 구현하였습니다.
마지막으로 common은 기타 유틸리티 성향을 가진 공통 클래스를 모았습니다.

## Di(Hilt)
```kotlin
@Module
@InstallIn(SingletonComponent::class)
class ApiModule {

	@Provides
	fun provideBaseUrlV2() = NetConstant.API_URL_V2

	@Singleton
	@Provides
	fun provideOkHttpClient() : OkHttpClient {
		val loggingInterceptor = HttpLoggingInterceptor()
		loggingInterceptor.level = HttpLoggingInterceptor.Level.BODY

		return OkHttpClient.Builder()
			.addInterceptor(loggingInterceptor)
			.build()
	}

	@Singleton
	@Provides
	fun provideRetrofit(okHttpClient: OkHttpClient, baseUrlV2: String, jsonMapper: JsonMapper) : Retrofit {
		return Retrofit.Builder()
			.client(okHttpClient)
			.baseUrl(baseUrlV2)
			.addConverterFactory(JacksonConverterFactory.create(jsonMapper))
			.build()
	}

	@Singleton
	@Provides
	fun provideJsonMapper(): JsonMapper {
		return jsonMapper {
			configure(DeserializationFeature.FAIL_ON_UNKNOWN_PROPERTIES, false)
			configure(DeserializationFeature.ACCEPT_EMPTY_STRING_AS_NULL_OBJECT, true)
			addModule(
				KotlinModule.Builder()
					.configure(KotlinFeature.NullIsSameAsDefault, true)
					.build()
			)
		}
	}

	@Singleton
	@Provides
	fun provideUsersJoinLoginService(retrofit: Retrofit): UsersJoinLoginService {
		return retrofit.create(UsersJoinLoginService::class.java)
	}
}
```
1. @Module : Hilt 모듈 생성을 위한 어노테이션 입니다.
2. @InstallIn(SingletonComponent::class) : Application이 살아있는 동안 유지할 컴포넌트로 선언하기 위한 어노테이션 입니다.
3. @Provides : DI 주입 시 생성할 오브젝트를 만들기 위한 Function에 사용합니다.
4. @Singleton : InstallIn의 선언에 따라 Singleton으로 유지할 오브젝트일 경우 사용 합니다.

Hilt를 통해 편리하게 Retofit 구현체를 만들어서 각각 Layer에서 필요할 경우 Singleton으로 사용하도록 하여, 메모리 낭비를 최소화 하였습니다.
단 Hilt로 Module 구현 시 주의해야할 부분은 

**"Function 이름이 다르더라도, 반환 오브젝트 클래스가 동일하면 안됩니다."**

이유는 주입 시 어떤 오브젝트를 주입해야할지 알 수 없기 때문 입니다.
(이로인해 아쉽게도 API 서버 URL이 여러개일 경우 이 방식으로는 커버하기 어렵습니다.)

## Data Layer
### 1) Source
Retrofit2 라이브러리를 통해서 만든 API 호출 클래스 입니다.   
Source 패키지 안에는 이렇게 API 호출 혹은 Local DB 기능을 구현해 놓습니다.  
```kotlin
interface UsersJoinLoginService {
	@POST("/api/v1/users/signin")
	suspend fun postSignIn(@Body data: SignRequestDto) : Response<SignInResponseDto>
}
```
  
### 2) Repository

```kotlin
interface UsersJoinLoginRepository {
	suspend fun postSignIn(data: SignRequestDto): Response<SignInResponseDto>
}
```
잠시 Domain Layer에 있는 Repository Interface를 먼저 넣었습니다.  
Domain Layer에서 선언한 Interface를 가지고 아래와 같이 실제 구현을 합니다.  
  
```kotlin
class UsersJoinLoginRepositoryImpl @Inject constructor(
	private val service: UsersJoinLoginService
) : UsersJoinLoginRepository {

	override suspend fun postSignIn(data: SignRequestDto): Response<SignInResponseDto> {
		return service.postSignIn(data)
	}
}
```
이렇게 할 경우 협업하는데 있어서 장점이 있습니다.
1. Repository Interface를 통해 구조도를 한번에 확인할 수 있습니다.
2. Data Layer와 Domain Layer간의 의존성을 최소화 할 수 있습니다.

다만 Repository Interface와 Repository 구현체가 1:1로 구성되기 때문에 잘못할 경우 보일러 플레이트 코드만 생산할 수 있습니다.  

추가로 @Inject는 의존성 주입을 위해 사용되었습니다.

### 3) Entity
API 호출 혹은 Local DB 기능 구현 시 사용되는 데이터 모델 클래스 입니다.  
Jackson 라이브러리를 사용해서 구현했습니다.
![](/assets/2021-10-20-Retrofit-Hilt-Part1/entity.png)  


## Domain Layer
### 1) Repository
위에서도 언급한 Repository Interface 입니다.
```kotlin
interface UsersJoinLoginRepository {
	suspend fun postSignIn(data: SignRequestDto): Response<SignInResponseDto>
}
```

### 2) Model
처음 공부를 했을 때   
**"왜 Data Layer에도 Entity가 있는데 굳이 Domain Layer에도 있는 것인가?"**  
란 의문이 들었습니다.  
  
실제로 구현해보면서 느낀 것은 실제로 적용할 때는 Entity와 Model을 혼용해서 사용할 것 같습니다.  
이유는 중간에 Mapper 클래스를 통해서 매번 Entity 데이터를 Model로 변환하는 과정은 필요 이상으로 피로감을 줄 것 같았습니다.  
  
그래서 Model은 Presentation에 전달 시 필요한 경우에만 만들고 일반적으로 Presentation은 Entity를 접근하게 하는 것이 개발을 하는 것에 있어서는 빠르게 대응이 될 것으로 생각하였습니다.
  
기존에 다른 참고 예제들을 보면 API 상태 변화 및 결과를 Presentation에 전달할 때 Model을 사용하는 것을 참고하여 만들었습니다.  

```kotlin
data class ApiResult<out T>(val status: Status, val code: String?, val message: String?, val data: T?, val exception: Exception?) {

	enum class Status {
		SUCCESS,
		API_ERROR,
		NETWORK_ERROR,
		LOADING
	}

	companion object {
		fun <T> success(code: String, data: T?): ApiResult<T> {
			return ApiResult(Status.SUCCESS, code, "", data, null)
		}

		fun <T> error(code: String, message: String): ApiResult<T> {
			return ApiResult(Status.API_ERROR, URLDecoder.decode(code, "UTF-8"), URLDecoder.decode(message, "UTF-8"), null, null)
		}

		fun <T> error(exception: Exception?): ApiResult<T> {
			return ApiResult(Status.NETWORK_ERROR, null, null, null, exception)
		}

		fun <T> loading(): ApiResult<T> {
			return ApiResult(Status.LOADING, null, null, null, null)
		}
	}

	override fun toString(): String {
		return "Result(status=$status, code=$code, message=$message, data=$data, error=$exception)"
	}
}
```

### 3) UseCase
어떻게 보면 제일 중요한 UseCase 입니다.  
실제 비지니스 로직이 담겨져 있으며 여러 역할을 합니다.  
1. Presentation에 상태 변화 및 값 전달
2. Data Layer에서 API 기능 수행 및 Local DB 기능 수행
3. Data Layer에서 만들어진 Data를 Presentation에 전달하기 위한 데이터 가공 

이 역할들을 구현할 때 참고한 자료에서는 다양한 방법들이 있었습니다. 
1. [RxKotlin](https://github.com/ReactiveX/RxKotlin)
2. [Coroutine](https://developer.android.com/kotlin/coroutines?hl=ko)
3. [Coroutine Flow](https://developer.android.com/kotlin/flow?hl=ko)

이왕 하는 것 차라리 가장 최근에 나온 Coroutine Flow 방식을 사용하고 싶어서 결국 Flow를 채택했습니다.

해당 예제 코드에서는 Base Class인 BaseRepository에서 데이터 조회 성공 여부에 따라 변환하는 Mapper Function을 사용하고 있습니다.
```kotlin
class UsersJoinLoginUseCase @Inject constructor(private val repository: UsersJoinLoginRepository) : BaseRepository() {

	suspend fun postSignIn(data: SignRequestDto) : Flow<ApiResult<SignInResponseDto>> {
		return flow {
			emit(ApiResult.loading())
			val result = getResponseV2(repository.postSignIn(data))
			emit(result)
		}
	}
}
```

```kotlin
open class BaseRepository {

	internal suspend fun <T> getResponse(response: Response<T>): ApiResult<T> {
		return try {
			if (response.isSuccessful) {
				return ApiResult.success("000000", response.body())
			} else {
				val code = response.headers()[MetaConstant.RESULT_CODE] ?: ""
				val message = response.headers()[MetaConstant.RESULT_MESSAGE] ?: ""
				ApiResult.error(code, message)
			}
		} catch (e: Exception) {
			ApiResult.error(e)
		}
	}
}
```

## 결론
해당 내용에 대해 공부하면서 장단점을 정리하였습니다.  

### 장점
1. Layer를 구분하고 각자 의존성을 최소화 하고, 
2. 역할을 명확히 구분해서 중복 역할을 미연에 방지
3. Coroutine을 활용해서 Thread 구현이 쉬움
4. 테스트 코드 작성이 수월함
5. MVVM 구현 시 편리함

### 단점
1. Retrofit2, Hilt, Coroutine 라이브러리를 학습 필요
2. Clean Architecture 룰에 맞추다보면, 보일러 플레이트 코드가 증가될 수 있음

아직까진 제가 이해한 내용이 부족할 수 있습니다.  
혹시라도 잘못된 점이 있다면 언제든지 조언 부탁드립니다 :)
Part 2 에서는 위 방식으로 구현했을 때 DI로 주입한 오브젝트들의 재사용성과 메모리 효율성에 대해 알아보려고 합니다.
감사합니다.

https://narendrasinhdodiya.medium.com/android-architecture-mvvm-with-coroutines-retrofit-hilt-kotlin-flow-room-48e67ca3b2c8  
https://namget.tistory.com/entry/%EC%95%88%EB%93%9C%EB%A1%9C%EC%9D%B4%EB%93%9C-Clean-Architecture  
https://leveloper.tistory.com/205  
https://itnext.io/android-architecture-hilt-mvvm-kotlin-coroutines-live-data-room-and-retrofit-ft-8b746cab4a06