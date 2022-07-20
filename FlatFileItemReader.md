# FlatFileItemReader
DB가 아닌 Reasouce에서 데이터를 읽어올 수 있도록 구현된 구현체

### 종류
- FileSystemResource, InputStreamResource, UrlResource, ByteArrayResource 등등

### Sample
```
@Bean
@StepScope

public FlatFileItemReader<UserDto> userItemReader() {
	return new FlatFileItemReaderBuilder<UserDto>()
		.name("userItemReader")
		.resource(new FileSystemResource(filePath + fileName))
		.delimited().delimiter("|")
		.names(new String[] {"userId", "passwd", "name","intValue"})
		.targetType(UserDto.class)
		.recordSeparatorPolicy(new SimpleRecordSeparatorPolicy() {
			@Override
			public String postProcess(String record) {
				return record.trim();
			}
		})
		.build();
}
```
- resource\
지정한 파일에서 데이터를 읽어올 수 있도록 resource 설정
- delimited\
한 라인에서 각각의 컬럼을 어떤 구분자로 구분할지..
- names\
파일 내 각각의 컬럼명 지정

|userId|passwd|name|intValue|매핑에러|
|------|---|---|---|---|
|테스트1|테스트2|테스트3|테스트3|---|
|테스트1|테스트2|테스트3|테스트3|---|
|테스트1|테스트2|테스트3|테스트3|error|

- targetType\
매핑되서 반환될 dto를 지정

```
@Getter
@Setter
public class UserDto {

    String userId;
    String passwd;
    String name;
    Integer intValue;

}
```

- recordSeparatorPolicy\
읽어온 라인의 추가 작업(postProcess)을 지정
앞뒤공백제거등...

### Sample
```
    private InputStream is;
    
    @Bean
    public Step fileStep() {
        return stepBuilderFactory.get("fileStep")
                .<UserDto, TbUser> chunk(10)
                .reader(userItemReader())
		***
                .faultTolerant()
                .skip(FlatFileParseException.class)
                .skipLimit(3)
		***
                .processor(userItemProcessor())
                .writer(userItemWriter())
                .listener(fileStepListener())
                .build();
    }

    @Bean
    public StepExecutionListener fileStepListener() {
        return new StepExecutionListenerSupport() {

            @SneakyThrows
            @Override
            public void beforeStep(StepExecution stepExecution) {
                is = new FileInputStream(filePath + fileName);
            }

            @Override
            public ExitStatus afterStep(StepExecution stepExecution) {
                return super.afterStep(stepExecution);
            }
        };
    }

    @Bean
    @StepScope
    public FlatFileItemReader<UserDto> userItemReader() {

        return new FlatFileItemReaderBuilder<UserDto>()
                .name("userItemReader")
                .resource(new InputStreamResource(is))
                .delimited().delimiter("|")
                .names(new String[] {"userId", "passwd", "name","intValue"})
                .targetType(UserDto.class)
                .recordSeparatorPolicy(new SimpleRecordSeparatorPolicy() {
                    @Override
                    public String postProcess(String record) {
                        return record.trim();
                    }
                })
                .build();
    }
```
- faultTolerant \
skip지정(장애 허용)이 가능
- skip\
스킵할 예외를 명시
- skipLimit\
에서 스킵할 최대 횟수를 지정

여기서는 스킵할 예외를 FlatFileParseException으로, 스킵 최대 횟수를 3으로 지정하였다.\
스킵 최대 횟수를 넘어서면 스킵횟수 초과 익셉션이 발생하게 된다.

listener에 SkipListenerSupport 의 onSkipInRead를 구현해주면, 스킵된 예외의 log 발생
