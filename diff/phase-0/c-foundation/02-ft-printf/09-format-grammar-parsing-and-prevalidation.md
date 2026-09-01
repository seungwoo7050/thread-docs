# 포맷 문법 해석과 선검증

## `feat(parser): 포맷 필드 모델과 해석기 추가`

diff --git a/Makefile b/Makefile
index 092971e..210a17a 100644
--- a/Makefile
+++ b/Makefile
@@ -8,7 +8,8 @@ ARFLAGS := rcs
 RM := rm -f
 
 SRC := src/ft_printf.c \
-	src/ft_output.c
+	src/ft_output.c \
+	src/ft_parse.c
 OBJ := $(SRC:.c=.o)
 HEADER := include/ft_printf.h src/ft_printf_internal.h
 
diff --git a/src/ft_parse.c b/src/ft_parse.c
new file mode 100644
index 0000000..72f883d
--- /dev/null
+++ b/src/ft_parse.c
@@ -0,0 +1,74 @@
+#include "ft_printf_internal.h"
+
+#include <limits.h>
+
+static int	ft_is_digit(char c)
+{
+	return (c >= '0' && c <= '9');
+}
+
+static int	ft_flag_value(char c)
+{
+	if (c == '-')
+		return (FT_FLAG_LEFT);
+	if (c == '0')
+		return (FT_FLAG_ZERO);
+	if (c == '#')
+		return (FT_FLAG_HASH);
+	if (c == ' ')
+		return (FT_FLAG_SPACE);
+	if (c == '+')
+		return (FT_FLAG_PLUS);
+	return (0);
+}
+
+static int	ft_parse_decimal(const char **format, int *value)
+{
+	int	digit;
+
+	while (ft_is_digit(**format))
+	{
+		digit = **format - '0';
+		if (*value > (INT_MAX - digit) / 10)
+			return (-1);
+		*value = *value * 10 + digit;
+		(*format)++;
+	}
+	return (0);
+}
+
+void	ft_printf_init_format(t_format *fmt)
+{
+	fmt->flags = 0;
+	fmt->width = 0;
+	fmt->precision = 0;
+	fmt->has_precision = 0;
+	fmt->spec = '\0';
+}
+
+const char	*ft_printf_parse(const char *format, t_format *fmt)
+{
+	int	flag;
+
+	ft_printf_init_format(fmt);
+	flag = ft_flag_value(*format);
+	while (flag)
+	{
+		fmt->flags |= flag;
+		format++;
+		flag = ft_flag_value(*format);
+	}
+	if (ft_parse_decimal(&format, &fmt->width) < 0)
+		return (0);
+	if (*format == '.')
+	{
+		fmt->has_precision = 1;
+		format++;
+		if (ft_parse_decimal(&format, &fmt->precision) < 0)
+			return (0);
+	}
+	fmt->spec = *format;
+	if (*format)
+		format++;
+	return (format);
+}
diff --git a/src/ft_printf_internal.h b/src/ft_printf_internal.h
index 524c99a..e4c6e47 100644
--- a/src/ft_printf_internal.h
+++ b/src/ft_printf_internal.h
@@ -5,6 +5,12 @@
 
 # include <stddef.h>
 
+# define FT_FLAG_LEFT 1
+# define FT_FLAG_ZERO 2
+# define FT_FLAG_HASH 4
+# define FT_FLAG_SPACE 8
+# define FT_FLAG_PLUS 16
+
 typedef struct s_printf
 {
 	int	fd;
@@ -12,8 +18,19 @@ typedef struct s_printf
 	int	error;
 }	t_printf;
 
+typedef struct s_format
+{
+	int		flags;
+	int		width;
+	int		precision;
+	int		has_precision;
+	char	spec;
+}	t_format;
+
 void		ft_printf_init(t_printf *ctx, int fd);
 int			ft_printf_write(t_printf *ctx, const char *buffer, size_t length);
 int			ft_printf_putchar(t_printf *ctx, char c);
+void		ft_printf_init_format(t_format *fmt);
+const char	*ft_printf_parse(const char *format, t_format *fmt);
 
 #endif


## `feat(core): 포맷 필드 해석을 출력 루프에 연결`

diff --git a/src/ft_printf.c b/src/ft_printf.c
index fd0e7bd..309fd28 100644
--- a/src/ft_printf.c
+++ b/src/ft_printf.c
@@ -6,6 +6,7 @@ int	ft_printf(const char *format, ...)
 {
 	va_list	args;
 	t_printf	ctx;
+	t_format	fmt;
 
 	(void)args;
 	if (format == 0)
@@ -20,6 +21,20 @@ int	ft_printf(const char *format, ...)
 				break ;
 			format += 2;
 		}
+		else if (*format == '%')
+		{
+			format = ft_printf_parse(format + 1, &fmt);
+			if (format == 0)
+			{
+				ctx.error = 1;
+				break ;
+			}
+			if (ft_printf_putchar(&ctx, '%') < 0)
+				break ;
+			if (fmt.spec != '\0' && fmt.spec != '%'
+				&& ft_printf_putchar(&ctx, fmt.spec) < 0)
+				break ;
+		}
 		else
 		{
 			if (ft_printf_putchar(&ctx, *format) < 0)


## `feat(flags): 숫자 플래그 우선순위 정규화`

diff --git a/src/ft_hex.c b/src/ft_hex.c
index 5dda872..d0fd0f8 100644
--- a/src/ft_hex.c
+++ b/src/ft_hex.c
@@ -74,7 +74,17 @@ static int	ft_write_hex(t_printf *ctx, t_format *fmt, const char *prefix,
 
 int	ft_printf_print_hex(t_printf *ctx, t_format *fmt, unsigned int number)
 {
-	return (ft_write_hex(ctx, fmt, "", (unsigned long)number));
+	const char	*prefix;
+
+	prefix = "";
+	if ((fmt->flags & FT_FLAG_HASH) && number != 0)
+	{
+		if (fmt->spec == 'X')
+			prefix = "0X";
+		else
+			prefix = "0x";
+	}
+	return (ft_write_hex(ctx, fmt, prefix, (unsigned long)number));
 }
 
 int	ft_printf_print_pointer(t_printf *ctx, t_format *fmt, void *pointer)
diff --git a/src/ft_number.c b/src/ft_number.c
index c89be26..80fcb42 100644
--- a/src/ft_number.c
+++ b/src/ft_number.c
@@ -70,11 +70,17 @@ static int	ft_write_decimal(t_printf *ctx, t_format *fmt,
 int	ft_printf_print_signed(t_printf *ctx, t_format *fmt, int number)
 {
 	long	value;
+	const char	*prefix;
 
 	value = (long)number;
 	if (value < 0)
 		return (ft_write_decimal(ctx, fmt, "-", (unsigned long)(-value)));
-	return (ft_write_decimal(ctx, fmt, "", (unsigned long)value));
+	prefix = "";
+	if (fmt->flags & FT_FLAG_PLUS)
+		prefix = "+";
+	else if (fmt->flags & FT_FLAG_SPACE)
+		prefix = " ";
+	return (ft_write_decimal(ctx, fmt, prefix, (unsigned long)value));
 }
 
 int	ft_printf_print_unsigned(t_printf *ctx, t_format *fmt, unsigned int number)
diff --git a/src/ft_parse.c b/src/ft_parse.c
index 72f883d..95c2c9f 100644
--- a/src/ft_parse.c
+++ b/src/ft_parse.c
@@ -46,6 +46,14 @@ void	ft_printf_init_format(t_format *fmt)
 	fmt->spec = '\0';
 }
 
+static void	ft_normalize_flags(t_format *fmt)
+{
+	if (fmt->flags & FT_FLAG_LEFT)
+		fmt->flags &= ~FT_FLAG_ZERO;
+	if (fmt->flags & FT_FLAG_PLUS)
+		fmt->flags &= ~FT_FLAG_SPACE;
+}
+
 const char	*ft_printf_parse(const char *format, t_format *fmt)
 {
 	int	flag;
@@ -70,5 +78,6 @@ const char	*ft_printf_parse(const char *format, t_format *fmt)
 	fmt->spec = *format;
 	if (*format)
 		format++;
+	ft_normalize_flags(fmt);
 	return (format);
 }


## `fix(format): 지원 문법과 전체 출력 크기 선검증`

diff --git a/Makefile b/Makefile
index 730b260..552c46d 100644
--- a/Makefile
+++ b/Makefile
@@ -10,6 +10,7 @@ RM := rm -f
 SRC := src/ft_printf.c \
 	src/ft_output.c \
 	src/ft_parse.c \
+	src/ft_measure.c \
 	src/ft_dispatch.c \
 	src/ft_text.c \
 	src/ft_numeric_layout.c \
diff --git a/src/ft_measure.c b/src/ft_measure.c
new file mode 100644
index 0000000..77e844d
--- /dev/null
+++ b/src/ft_measure.c
@@ -0,0 +1,188 @@
+#include "ft_printf_internal.h"
+
+#include <limits.h>
+#include <stdint.h>
+
+static int	ft_is_supported_specifier(char spec)
+{
+	return (spec == 'c' || spec == 's' || spec == 'd' || spec == 'i'
+		|| spec == 'u' || spec == 'x' || spec == 'X' || spec == 'p'
+		|| spec == '%');
+}
+
+static int	ft_add_length(size_t *total, size_t amount)
+{
+	if (amount > (size_t)INT_MAX || *total > (size_t)INT_MAX - amount)
+		return (-1);
+	*total += amount;
+	return (0);
+}
+
+static int	ft_decimal_length(unsigned long number)
+{
+	int	length;
+
+	length = 1;
+	while (number >= 10)
+	{
+		number /= 10;
+		length++;
+	}
+	return (length);
+}
+
+static int	ft_hex_length(unsigned long number)
+{
+	int	length;
+
+	length = 1;
+	while (number >= 16)
+	{
+		number /= 16;
+		length++;
+	}
+	return (length);
+}
+
+static int	ft_measure_numeric(t_format *fmt, int prefix_length,
+		int digit_length, int is_zero, size_t *length)
+{
+	size_t	content_length;
+
+	if (fmt->has_precision && fmt->precision == 0 && is_zero)
+		digit_length = 0;
+	content_length = (size_t)digit_length;
+	if (fmt->has_precision && fmt->precision > digit_length)
+		content_length = (size_t)fmt->precision;
+	if (ft_add_length(&content_length, (size_t)prefix_length) < 0)
+		return (-1);
+	if ((size_t)fmt->width > content_length)
+		content_length = (size_t)fmt->width;
+	*length = content_length;
+	return (0);
+}
+
+static int	ft_measure_string(t_format *fmt, const char *string,
+		size_t *length)
+{
+	size_t	string_length;
+
+	if (string == 0)
+		string = "(null)";
+	string_length = 0;
+	while ((!fmt->has_precision
+			|| string_length < (size_t)fmt->precision)
+		&& string[string_length])
+	{
+		if (string_length == (size_t)INT_MAX)
+			return (-1);
+		string_length++;
+	}
+	if ((size_t)fmt->width > string_length)
+		string_length = (size_t)fmt->width;
+	*length = string_length;
+	return (0);
+}
+
+static int	ft_measure_signed(t_format *fmt, int number, size_t *length)
+{
+	long			value;
+	unsigned long	magnitude;
+	int				prefix_length;
+
+	value = (long)number;
+	prefix_length = 0;
+	if (value < 0)
+	{
+		prefix_length = 1;
+		magnitude = (unsigned long)(-(value + 1)) + 1;
+	}
+	else
+	{
+		magnitude = (unsigned long)value;
+		if (fmt->flags & (FT_FLAG_PLUS | FT_FLAG_SPACE))
+			prefix_length = 1;
+	}
+	return (ft_measure_numeric(fmt, prefix_length,
+			ft_decimal_length(magnitude), magnitude == 0, length));
+}
+
+static int	ft_measure_hex(t_format *fmt, unsigned long number,
+		int is_pointer, size_t *length)
+{
+	int	prefix_length;
+
+	prefix_length = 0;
+	if (is_pointer || ((fmt->flags & FT_FLAG_HASH) && number != 0))
+		prefix_length = 2;
+	return (ft_measure_numeric(fmt, prefix_length, ft_hex_length(number),
+			number == 0, length));
+}
+
+static int	ft_measure_conversion(t_format *fmt, va_list *args,
+		size_t *length)
+{
+	unsigned int	unsigned_number;
+
+	if (fmt->spec == 'c')
+	{
+		(void)va_arg(*args, int);
+		*length = 1;
+	}
+	else if (fmt->spec == 's')
+		return (ft_measure_string(fmt, va_arg(*args, char *), length));
+	else if (fmt->spec == 'd' || fmt->spec == 'i')
+		return (ft_measure_signed(fmt, va_arg(*args, int), length));
+	else if (fmt->spec == 'u')
+	{
+		unsigned_number = va_arg(*args, unsigned int);
+		return (ft_measure_numeric(fmt, 0,
+				ft_decimal_length((unsigned long)unsigned_number),
+				unsigned_number == 0, length));
+	}
+	else if (fmt->spec == 'x' || fmt->spec == 'X')
+		return (ft_measure_hex(fmt,
+				(unsigned long)va_arg(*args, unsigned int), 0, length));
+	else if (fmt->spec == 'p')
+		return (ft_measure_hex(fmt,
+				(unsigned long)(uintptr_t)va_arg(*args, void *), 1, length));
+	else
+		*length = 1;
+	if ((size_t)fmt->width > *length)
+		*length = (size_t)fmt->width;
+	return (0);
+}
+
+int	ft_printf_measure(const char *format, va_list *args)
+{
+	t_format	fmt;
+	size_t		conversion_length;
+	size_t		total;
+
+	total = 0;
+	while (*format)
+	{
+		if (*format == '%' && *(format + 1) == '%')
+		{
+			if (ft_add_length(&total, 1) < 0)
+				return (-1);
+			format += 2;
+		}
+		else if (*format == '%')
+		{
+			format = ft_printf_parse(format + 1, &fmt);
+			if (format == 0 || !ft_is_supported_specifier(fmt.spec)
+				|| ft_measure_conversion(&fmt, args,
+					&conversion_length) < 0
+				|| ft_add_length(&total, conversion_length) < 0)
+				return (-1);
+		}
+		else
+		{
+			if (ft_add_length(&total, 1) < 0)
+				return (-1);
+			format++;
+		}
+	}
+	return ((int)total);
+}
diff --git a/src/ft_printf.c b/src/ft_printf.c
index be2dedd..159a82b 100644
--- a/src/ft_printf.c
+++ b/src/ft_printf.c
@@ -3,13 +3,21 @@
 int	ft_printf(const char *format, ...)
 {
 	va_list	args;
+	va_list	measure_args;
 	t_printf	ctx;
 	t_format	fmt;
 
-	(void)args;
 	if (format == 0)
 		return (-1);
 	va_start(args, format);
+	va_copy(measure_args, args);
+	if (ft_printf_measure(format, &measure_args) < 0)
+	{
+		va_end(measure_args);
+		va_end(args);
+		return (-1);
+	}
+	va_end(measure_args);
 	ft_printf_init(&ctx, 1);
 	while (*format)
 	{
diff --git a/src/ft_printf_internal.h b/src/ft_printf_internal.h
index 5bb0aee..088ffcd 100644
--- a/src/ft_printf_internal.h
+++ b/src/ft_printf_internal.h
@@ -32,6 +32,7 @@ void		ft_printf_init(t_printf *ctx, int fd);
 int			ft_printf_write(t_printf *ctx, const char *buffer, size_t length);
 int			ft_printf_putchar(t_printf *ctx, char c);
 int			ft_printf_putnchar(t_printf *ctx, char c, int length);
+int			ft_printf_measure(const char *format, va_list *args);
 int			ft_printf_write_numeric_layout(t_printf *ctx, t_format *fmt,
 				const char *prefix, const char *digits, int digit_len,
 				int is_zero);


## `test(format): 잘못된 포맷의 무출력 실패 검증`

diff --git a/tests/test_ft_printf.c b/tests/test_ft_printf.c
index b5965d8..b4d38ce 100644
--- a/tests/test_ft_printf.c
+++ b/tests/test_ft_printf.c
@@ -108,6 +108,18 @@ static void	check_case(int line, const char *format, const char *expected,
 		actual_ret); \
 } while (0)
 
+#define EXPECT_FORMAT_ERROR(FORMAT, ...) do { \
+	char		actual[16]; \
+	t_capture	capture; \
+	int			actual_ret; \
+	ssize_t		actual_len; \
+	capture_begin(&capture, __LINE__); \
+	actual_ret = ft_printf(FORMAT, ##__VA_ARGS__); \
+	actual_len = capture_end(&capture, actual, sizeof(actual), __LINE__); \
+	if (actual_ret != -1 || actual_len != 0) \
+		fail_test(__LINE__, "invalid format produced output"); \
+} while (0)
+
 #define EXPECT_OUTPUT(EXPECTED, FORMAT, ...) do { \
 	char		actual[4096]; \
 	t_capture	capture; \
@@ -172,6 +184,14 @@ static void	run_parser_boundary_cases(void)
 {
 	expect_field_error(__LINE__, "%2147483648d");
 	expect_field_error(__LINE__, "%.2147483648d");
+	EXPECT_FORMAT_ERROR("prefix:%2147483648d", 1);
+	EXPECT_FORMAT_ERROR("prefix:%q", 1);
+	EXPECT_FORMAT_ERROR("prefix:%");
+	EXPECT_FORMAT_ERROR("value:%d bad:%q", 7, 1);
+	EXPECT_FORMAT_ERROR("x%2147483647d", 1);
+	EXPECT_FORMAT_ERROR("%2147483647dX", 1);
+	EXPECT_FORMAT_ERROR("%+.2147483647d", 1);
+	EXPECT_FORMAT_ERROR("%#.2147483647x", 1u);
 }
 
 static void	run_numeric_layout_cases(void)
