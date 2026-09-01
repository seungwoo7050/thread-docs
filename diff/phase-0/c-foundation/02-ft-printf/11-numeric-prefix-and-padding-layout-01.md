# 숫자 접두사와 패딩 배치

## `feat(decimal): 부호 있는·없는 10진수 출력 추가`

diff --git a/Makefile b/Makefile
index fb35e4e..bdd8f04 100644
--- a/Makefile
+++ b/Makefile
@@ -11,7 +11,8 @@ SRC := src/ft_printf.c \
 	src/ft_output.c \
 	src/ft_parse.c \
 	src/ft_dispatch.c \
-	src/ft_text.c
+	src/ft_text.c \
+	src/ft_number.c
 OBJ := $(SRC:.c=.o)
 HEADER := include/ft_printf.h src/ft_printf_internal.h
 
diff --git a/src/ft_dispatch.c b/src/ft_dispatch.c
index dede10c..9bb1e13 100644
--- a/src/ft_dispatch.c
+++ b/src/ft_dispatch.c
@@ -6,6 +6,11 @@ int	ft_printf_dispatch(t_printf *ctx, t_format *fmt, va_list *args)
 		return (ft_printf_print_char(ctx, fmt, va_arg(*args, int)));
 	if (fmt->spec == 's')
 		return (ft_printf_print_string(ctx, fmt, va_arg(*args, char *)));
+	if (fmt->spec == 'd' || fmt->spec == 'i')
+		return (ft_printf_print_signed(ctx, fmt, va_arg(*args, int)));
+	if (fmt->spec == 'u')
+		return (ft_printf_print_unsigned(ctx, fmt,
+				va_arg(*args, unsigned int)));
 	if (fmt->spec == '%')
 		return (ft_printf_print_percent(ctx, fmt));
 	if (ft_printf_putchar(ctx, '%') < 0)
diff --git a/src/ft_number.c b/src/ft_number.c
new file mode 100644
index 0000000..2290926
--- /dev/null
+++ b/src/ft_number.c
@@ -0,0 +1,44 @@
+#include "ft_printf_internal.h"
+
+static int	ft_print_unsigned_digits(t_printf *ctx, unsigned long number)
+{
+	char	buffer[20];
+	int		index;
+
+	index = 0;
+	if (number == 0)
+		buffer[index++] = '0';
+	while (number > 0)
+	{
+		buffer[index++] = (char)('0' + number % 10);
+		number /= 10;
+	}
+	while (index > 0)
+	{
+		index--;
+		if (ft_printf_putchar(ctx, buffer[index]) < 0)
+			return (-1);
+	}
+	return (0);
+}
+
+int	ft_printf_print_signed(t_printf *ctx, t_format *fmt, int number)
+{
+	long	value;
+
+	(void)fmt;
+	value = (long)number;
+	if (value < 0)
+	{
+		if (ft_printf_putchar(ctx, '-') < 0)
+			return (-1);
+		value = -value;
+	}
+	return (ft_print_unsigned_digits(ctx, (unsigned long)value));
+}
+
+int	ft_printf_print_unsigned(t_printf *ctx, t_format *fmt, unsigned int number)
+{
+	(void)fmt;
+	return (ft_print_unsigned_digits(ctx, (unsigned long)number));
+}
diff --git a/src/ft_printf_internal.h b/src/ft_printf_internal.h
index 70695ad..c2da9dc 100644
--- a/src/ft_printf_internal.h
+++ b/src/ft_printf_internal.h
@@ -38,5 +38,8 @@ int			ft_printf_print_char(t_printf *ctx, t_format *fmt, int c);
 int			ft_printf_print_string(t_printf *ctx, t_format *fmt,
 				const char *string);
 int			ft_printf_print_percent(t_printf *ctx, t_format *fmt);
+int			ft_printf_print_signed(t_printf *ctx, t_format *fmt, int number);
+int			ft_printf_print_unsigned(t_printf *ctx, t_format *fmt,
+				unsigned int number);
 
 #endif


## `feat(hex): 16진수와 포인터 출력 추가`

diff --git a/Makefile b/Makefile
index bdd8f04..fbaddff 100644
--- a/Makefile
+++ b/Makefile
@@ -12,7 +12,8 @@ SRC := src/ft_printf.c \
 	src/ft_parse.c \
 	src/ft_dispatch.c \
 	src/ft_text.c \
-	src/ft_number.c
+	src/ft_number.c \
+	src/ft_hex.c
 OBJ := $(SRC:.c=.o)
 HEADER := include/ft_printf.h src/ft_printf_internal.h
 
diff --git a/src/ft_dispatch.c b/src/ft_dispatch.c
index 9bb1e13..b7b9a7a 100644
--- a/src/ft_dispatch.c
+++ b/src/ft_dispatch.c
@@ -11,6 +11,10 @@ int	ft_printf_dispatch(t_printf *ctx, t_format *fmt, va_list *args)
 	if (fmt->spec == 'u')
 		return (ft_printf_print_unsigned(ctx, fmt,
 				va_arg(*args, unsigned int)));
+	if (fmt->spec == 'x' || fmt->spec == 'X')
+		return (ft_printf_print_hex(ctx, fmt, va_arg(*args, unsigned int)));
+	if (fmt->spec == 'p')
+		return (ft_printf_print_pointer(ctx, fmt, va_arg(*args, void *)));
 	if (fmt->spec == '%')
 		return (ft_printf_print_percent(ctx, fmt));
 	if (ft_printf_putchar(ctx, '%') < 0)
diff --git a/src/ft_hex.c b/src/ft_hex.c
new file mode 100644
index 0000000..132d785
--- /dev/null
+++ b/src/ft_hex.c
@@ -0,0 +1,41 @@
+#include "ft_printf_internal.h"
+
+#include <stdint.h>
+
+static int	ft_print_base(t_printf *ctx, unsigned long number, const char *digits)
+{
+	char	buffer[2 + sizeof(unsigned long) * 2];
+	int		index;
+
+	index = 0;
+	if (number == 0)
+		buffer[index++] = '0';
+	while (number > 0)
+	{
+		buffer[index++] = digits[number % 16];
+		number /= 16;
+	}
+	while (index > 0)
+	{
+		index--;
+		if (ft_printf_putchar(ctx, buffer[index]) < 0)
+			return (-1);
+	}
+	return (0);
+}
+
+int	ft_printf_print_hex(t_printf *ctx, t_format *fmt, unsigned int number)
+{
+	if (fmt->spec == 'X')
+		return (ft_print_base(ctx, (unsigned long)number, "0123456789ABCDEF"));
+	return (ft_print_base(ctx, (unsigned long)number, "0123456789abcdef"));
+}
+
+int	ft_printf_print_pointer(t_printf *ctx, t_format *fmt, void *pointer)
+{
+	(void)fmt;
+	if (ft_printf_write(ctx, "0x", 2) < 0)
+		return (-1);
+	return (ft_print_base(ctx, (unsigned long)(uintptr_t)pointer,
+			"0123456789abcdef"));
+}
diff --git a/src/ft_printf_internal.h b/src/ft_printf_internal.h
index c2da9dc..4bc3bb9 100644
--- a/src/ft_printf_internal.h
+++ b/src/ft_printf_internal.h
@@ -41,5 +41,9 @@ int			ft_printf_print_percent(t_printf *ctx, t_format *fmt);
 int			ft_printf_print_signed(t_printf *ctx, t_format *fmt, int number);
 int			ft_printf_print_unsigned(t_printf *ctx, t_format *fmt,
 				unsigned int number);
+int			ft_printf_print_hex(t_printf *ctx, t_format *fmt,
+				unsigned int number);
+int			ft_printf_print_pointer(t_printf *ctx, t_format *fmt,
+				void *pointer);
 
 #endif


## `feat(decimal): 10진수 너비와 정렬 적용`

diff --git a/src/ft_number.c b/src/ft_number.c
index 2290926..fa6aaee 100644
--- a/src/ft_number.c
+++ b/src/ft_number.c
@@ -1,24 +1,50 @@
 #include "ft_printf_internal.h"
 
-static int	ft_print_unsigned_digits(t_printf *ctx, unsigned long number)
+static int	ft_decimal_digits(char *buffer, unsigned long number)
 {
-	char	buffer[20];
-	int		index;
+	char	reversed[20];
+	int		length;
 
-	index = 0;
+	length = 0;
 	if (number == 0)
-		buffer[index++] = '0';
+		reversed[length++] = '0';
 	while (number > 0)
 	{
-		buffer[index++] = (char)('0' + number % 10);
+		reversed[length++] = (char)('0' + number % 10);
 		number /= 10;
 	}
-	while (index > 0)
+	number = 0;
+	while (number < (unsigned long)length)
 	{
-		index--;
-		if (ft_printf_putchar(ctx, buffer[index]) < 0)
-			return (-1);
+		buffer[number] = reversed[length - 1 - number];
+		number++;
 	}
+	return (length);
+}
+
+static int	ft_write_decimal(t_printf *ctx, t_format *fmt,
+		const char *prefix, unsigned long number)
+{
+	char	digits[20];
+	int		digit_len;
+	int		prefix_len;
+	int		padding;
+
+	digit_len = ft_decimal_digits(digits, number);
+	prefix_len = 0;
+	while (prefix[prefix_len])
+		prefix_len++;
+	padding = fmt->width - prefix_len - digit_len;
+	if (!(fmt->flags & FT_FLAG_LEFT)
+		&& ft_printf_putnchar(ctx, ' ', padding) < 0)
+		return (-1);
+	if (ft_printf_write(ctx, prefix, (size_t)prefix_len) < 0)
+		return (-1);
+	if (ft_printf_write(ctx, digits, (size_t)digit_len) < 0)
+		return (-1);
+	if ((fmt->flags & FT_FLAG_LEFT)
+		&& ft_printf_putnchar(ctx, ' ', padding) < 0)
+		return (-1);
 	return (0);
 }
 
@@ -26,19 +52,13 @@ int	ft_printf_print_signed(t_printf *ctx, t_format *fmt, int number)
 {
 	long	value;
 
-	(void)fmt;
 	value = (long)number;
 	if (value < 0)
-	{
-		if (ft_printf_putchar(ctx, '-') < 0)
-			return (-1);
-		value = -value;
-	}
-	return (ft_print_unsigned_digits(ctx, (unsigned long)value));
+		return (ft_write_decimal(ctx, fmt, "-", (unsigned long)(-value)));
+	return (ft_write_decimal(ctx, fmt, "", (unsigned long)value));
 }
 
 int	ft_printf_print_unsigned(t_printf *ctx, t_format *fmt, unsigned int number)
 {
-	(void)fmt;
-	return (ft_print_unsigned_digits(ctx, (unsigned long)number));
+	return (ft_write_decimal(ctx, fmt, "", (unsigned long)number));
 }


## `feat(hex): 16진수와 포인터 너비와 정렬 적용`

diff --git a/src/ft_hex.c b/src/ft_hex.c
index 132d785..c84ac42 100644
--- a/src/ft_hex.c
+++ b/src/ft_hex.c
@@ -2,40 +2,65 @@
 
 #include <stdint.h>
 
-static int	ft_print_base(t_printf *ctx, unsigned long number, const char *digits)
+static int	ft_hex_digits(char *buffer, unsigned long number, const char *base)
 {
-	char	buffer[2 + sizeof(unsigned long) * 2];
+	char	reversed[2 + sizeof(unsigned long) * 2];
+	int		length;
 	int		index;
 
-	index = 0;
+	length = 0;
 	if (number == 0)
-		buffer[index++] = '0';
+		reversed[length++] = '0';
 	while (number > 0)
 	{
-		buffer[index++] = digits[number % 16];
+		reversed[length++] = base[number % 16];
 		number /= 16;
 	}
-	while (index > 0)
+	index = 0;
+	while (index < length)
 	{
-		index--;
-		if (ft_printf_putchar(ctx, buffer[index]) < 0)
-			return (-1);
+		buffer[index] = reversed[length - 1 - index];
+		index++;
 	}
+	return (length);
+}
+
+static int	ft_write_hex(t_printf *ctx, t_format *fmt, const char *prefix,
+		unsigned long number)
+{
+	char	digits[2 + sizeof(unsigned long) * 2];
+	int		digit_len;
+	int		prefix_len;
+	int		padding;
+
+	if (fmt->spec == 'X')
+		digit_len = ft_hex_digits(digits, number, "0123456789ABCDEF");
+	else
+		digit_len = ft_hex_digits(digits, number, "0123456789abcdef");
+	prefix_len = 0;
+	while (prefix[prefix_len])
+		prefix_len++;
+	padding = fmt->width - prefix_len - digit_len;
+	if (!(fmt->flags & FT_FLAG_LEFT)
+		&& ft_printf_putnchar(ctx, ' ', padding) < 0)
+		return (-1);
+	if (ft_printf_write(ctx, prefix, (size_t)prefix_len) < 0)
+		return (-1);
+	if (ft_printf_write(ctx, digits, (size_t)digit_len) < 0)
+		return (-1);
+	if ((fmt->flags & FT_FLAG_LEFT)
+		&& ft_printf_putnchar(ctx, ' ', padding) < 0)
+		return (-1);
 	return (0);
 }
 
 int	ft_printf_print_hex(t_printf *ctx, t_format *fmt, unsigned int number)
 {
-	if (fmt->spec == 'X')
-		return (ft_print_base(ctx, (unsigned long)number, "0123456789ABCDEF"));
-	return (ft_print_base(ctx, (unsigned long)number, "0123456789abcdef"));
+	return (ft_write_hex(ctx, fmt, "", (unsigned long)number));
 }
 
 int	ft_printf_print_pointer(t_printf *ctx, t_format *fmt, void *pointer)
 {
-	(void)fmt;
-	if (ft_printf_write(ctx, "0x", 2) < 0)
-		return (-1);
-	return (ft_print_base(ctx, (unsigned long)(uintptr_t)pointer,
-			"0123456789abcdef"));
+	return (ft_write_hex(ctx, fmt, "0x",
+			(unsigned long)(uintptr_t)pointer));
 }


## `feat(numeric): 숫자 정밀도와 0 채움 적용`

diff --git a/src/ft_hex.c b/src/ft_hex.c
index c84ac42..5dda872 100644
--- a/src/ft_hex.c
+++ b/src/ft_hex.c
@@ -31,21 +31,39 @@ static int	ft_write_hex(t_printf *ctx, t_format *fmt, const char *prefix,
 	char	digits[2 + sizeof(unsigned long) * 2];
 	int		digit_len;
 	int		prefix_len;
+	int		zero_len;
 	int		padding;
+	char	pad_char;
 
 	if (fmt->spec == 'X')
 		digit_len = ft_hex_digits(digits, number, "0123456789ABCDEF");
 	else
 		digit_len = ft_hex_digits(digits, number, "0123456789abcdef");
+	if (fmt->has_precision && fmt->precision == 0 && number == 0)
+		digit_len = 0;
 	prefix_len = 0;
 	while (prefix[prefix_len])
 		prefix_len++;
-	padding = fmt->width - prefix_len - digit_len;
+	zero_len = 0;
+	if (fmt->has_precision && fmt->precision > digit_len)
+		zero_len = fmt->precision - digit_len;
+	padding = fmt->width - prefix_len - zero_len - digit_len;
+	pad_char = ' ';
+	if ((fmt->flags & FT_FLAG_ZERO) && !(fmt->flags & FT_FLAG_LEFT)
+		&& !fmt->has_precision)
+		pad_char = '0';
 	if (!(fmt->flags & FT_FLAG_LEFT)
+		&& pad_char == ' '
 		&& ft_printf_putnchar(ctx, ' ', padding) < 0)
 		return (-1);
 	if (ft_printf_write(ctx, prefix, (size_t)prefix_len) < 0)
 		return (-1);
+	if (!(fmt->flags & FT_FLAG_LEFT)
+		&& pad_char == '0'
+		&& ft_printf_putnchar(ctx, '0', padding) < 0)
+		return (-1);
+	if (ft_printf_putnchar(ctx, '0', zero_len) < 0)
+		return (-1);
 	if (ft_printf_write(ctx, digits, (size_t)digit_len) < 0)
 		return (-1);
 	if ((fmt->flags & FT_FLAG_LEFT)
diff --git a/src/ft_number.c b/src/ft_number.c
index fa6aaee..c89be26 100644
--- a/src/ft_number.c
+++ b/src/ft_number.c
@@ -4,6 +4,7 @@ static int	ft_decimal_digits(char *buffer, unsigned long number)
 {
 	char	reversed[20];
 	int		length;
+	int		index;
 
 	length = 0;
 	if (number == 0)
@@ -13,11 +14,11 @@ static int	ft_decimal_digits(char *buffer, unsigned long number)
 		reversed[length++] = (char)('0' + number % 10);
 		number /= 10;
 	}
-	number = 0;
-	while (number < (unsigned long)length)
+	index = 0;
+	while (index < length)
 	{
-		buffer[number] = reversed[length - 1 - number];
-		number++;
+		buffer[index] = reversed[length - 1 - index];
+		index++;
 	}
 	return (length);
 }
@@ -28,18 +29,36 @@ static int	ft_write_decimal(t_printf *ctx, t_format *fmt,
 	char	digits[20];
 	int		digit_len;
 	int		prefix_len;
+	int		zero_len;
 	int		padding;
+	char	pad_char;
 
 	digit_len = ft_decimal_digits(digits, number);
+	if (fmt->has_precision && fmt->precision == 0 && number == 0)
+		digit_len = 0;
 	prefix_len = 0;
 	while (prefix[prefix_len])
 		prefix_len++;
-	padding = fmt->width - prefix_len - digit_len;
+	zero_len = 0;
+	if (fmt->has_precision && fmt->precision > digit_len)
+		zero_len = fmt->precision - digit_len;
+	padding = fmt->width - prefix_len - zero_len - digit_len;
+	pad_char = ' ';
+	if ((fmt->flags & FT_FLAG_ZERO) && !(fmt->flags & FT_FLAG_LEFT)
+		&& !fmt->has_precision)
+		pad_char = '0';
 	if (!(fmt->flags & FT_FLAG_LEFT)
+		&& pad_char == ' '
 		&& ft_printf_putnchar(ctx, ' ', padding) < 0)
 		return (-1);
 	if (ft_printf_write(ctx, prefix, (size_t)prefix_len) < 0)
 		return (-1);
+	if (!(fmt->flags & FT_FLAG_LEFT)
+		&& pad_char == '0'
+		&& ft_printf_putnchar(ctx, '0', padding) < 0)
+		return (-1);
+	if (ft_printf_putnchar(ctx, '0', zero_len) < 0)
+		return (-1);
 	if (ft_printf_write(ctx, digits, (size_t)digit_len) < 0)
 		return (-1);
 	if ((fmt->flags & FT_FLAG_LEFT)


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


## `test(numeric): 접두사와 정밀도 배치 회귀 검증`

diff --git a/tests/test_ft_printf.c b/tests/test_ft_printf.c
index bb08d3f..77e7bcb 100644
--- a/tests/test_ft_printf.c
+++ b/tests/test_ft_printf.c
@@ -173,6 +173,19 @@ static void	run_parser_boundary_cases(void)
 	expect_field_error(__LINE__, "%.2147483648d");
 }
 
+static void	run_numeric_layout_cases(void)
+{
+	const char	*space_precision;
+
+	EXPECT_PRINTF("empty:'%#.0x' '%#.0X' '% .0d'", 0u, 0u, 0);
+	EXPECT_PRINTF("signed-zero:'%+08d'", 42);
+	EXPECT_PRINTF("hex-zero:'%#08x'", 42u);
+	EXPECT_PRINTF("hex-left-precision:'%-#10.4x'", 42u);
+	space_precision = "signed-space-precision:'% 08.5d'";
+	EXPECT_PRINTF(space_precision, 42);
+	EXPECT_PRINTF("hex-empty:'%#.0x'", 0u);
+}
+
 static void	run_error_cases(void)
 {
 	int	saved_stdout;
@@ -197,6 +210,7 @@ int	main(void)
 	run_core_cases();
 	run_bonus_cases();
 	run_parser_boundary_cases();
+	run_numeric_layout_cases();
 	run_error_cases();
 	dprintf(STDERR_FILENO, "ft_printf tests passed\n");
 	return (0);


## `refactor(output): 숫자 출력 배치 로직 통합`

diff --git a/Makefile b/Makefile
index 216772d..8a69095 100644
--- a/Makefile
+++ b/Makefile
@@ -12,6 +12,7 @@ SRC := src/ft_printf.c \
 	src/ft_parse.c \
 	src/ft_dispatch.c \
 	src/ft_text.c \
+	src/ft_numeric_layout.c \
 	src/ft_number.c \
 	src/ft_hex.c
 OBJ := $(SRC:.c=.o)
diff --git a/src/ft_hex.c b/src/ft_hex.c
index d0fd0f8..c1f9840 100644
--- a/src/ft_hex.c
+++ b/src/ft_hex.c
@@ -30,46 +30,13 @@ static int	ft_write_hex(t_printf *ctx, t_format *fmt, const char *prefix,
 {
 	char	digits[2 + sizeof(unsigned long) * 2];
 	int		digit_len;
-	int		prefix_len;
-	int		zero_len;
-	int		padding;
-	char	pad_char;
 
 	if (fmt->spec == 'X')
 		digit_len = ft_hex_digits(digits, number, "0123456789ABCDEF");
 	else
 		digit_len = ft_hex_digits(digits, number, "0123456789abcdef");
-	if (fmt->has_precision && fmt->precision == 0 && number == 0)
-		digit_len = 0;
-	prefix_len = 0;
-	while (prefix[prefix_len])
-		prefix_len++;
-	zero_len = 0;
-	if (fmt->has_precision && fmt->precision > digit_len)
-		zero_len = fmt->precision - digit_len;
-	padding = fmt->width - prefix_len - zero_len - digit_len;
-	pad_char = ' ';
-	if ((fmt->flags & FT_FLAG_ZERO) && !(fmt->flags & FT_FLAG_LEFT)
-		&& !fmt->has_precision)
-		pad_char = '0';
-	if (!(fmt->flags & FT_FLAG_LEFT)
-		&& pad_char == ' '
-		&& ft_printf_putnchar(ctx, ' ', padding) < 0)
-		return (-1);
-	if (ft_printf_write(ctx, prefix, (size_t)prefix_len) < 0)
-		return (-1);
-	if (!(fmt->flags & FT_FLAG_LEFT)
-		&& pad_char == '0'
-		&& ft_printf_putnchar(ctx, '0', padding) < 0)
-		return (-1);
-	if (ft_printf_putnchar(ctx, '0', zero_len) < 0)
-		return (-1);
-	if (ft_printf_write(ctx, digits, (size_t)digit_len) < 0)
-		return (-1);
-	if ((fmt->flags & FT_FLAG_LEFT)
-		&& ft_printf_putnchar(ctx, ' ', padding) < 0)
-		return (-1);
-	return (0);
+	return (ft_printf_write_numeric_layout(ctx, fmt, prefix, digits,
+			digit_len, number == 0));
 }
 
 int	ft_printf_print_hex(t_printf *ctx, t_format *fmt, unsigned int number)
diff --git a/src/ft_number.c b/src/ft_number.c
index 80fcb42..1b5bf74 100644
--- a/src/ft_number.c
+++ b/src/ft_number.c
@@ -28,43 +28,10 @@ static int	ft_write_decimal(t_printf *ctx, t_format *fmt,
 {
 	char	digits[20];
 	int		digit_len;
-	int		prefix_len;
-	int		zero_len;
-	int		padding;
-	char	pad_char;
 
 	digit_len = ft_decimal_digits(digits, number);
-	if (fmt->has_precision && fmt->precision == 0 && number == 0)
-		digit_len = 0;
-	prefix_len = 0;
-	while (prefix[prefix_len])
-		prefix_len++;
-	zero_len = 0;
-	if (fmt->has_precision && fmt->precision > digit_len)
-		zero_len = fmt->precision - digit_len;
-	padding = fmt->width - prefix_len - zero_len - digit_len;
-	pad_char = ' ';
-	if ((fmt->flags & FT_FLAG_ZERO) && !(fmt->flags & FT_FLAG_LEFT)
-		&& !fmt->has_precision)
-		pad_char = '0';
-	if (!(fmt->flags & FT_FLAG_LEFT)
-		&& pad_char == ' '
-		&& ft_printf_putnchar(ctx, ' ', padding) < 0)
-		return (-1);
-	if (ft_printf_write(ctx, prefix, (size_t)prefix_len) < 0)
-		return (-1);
-	if (!(fmt->flags & FT_FLAG_LEFT)
-		&& pad_char == '0'
-		&& ft_printf_putnchar(ctx, '0', padding) < 0)
-		return (-1);
-	if (ft_printf_putnchar(ctx, '0', zero_len) < 0)
-		return (-1);
-	if (ft_printf_write(ctx, digits, (size_t)digit_len) < 0)
-		return (-1);
-	if ((fmt->flags & FT_FLAG_LEFT)
-		&& ft_printf_putnchar(ctx, ' ', padding) < 0)
-		return (-1);
-	return (0);
+	return (ft_printf_write_numeric_layout(ctx, fmt, prefix, digits,
+			digit_len, number == 0));
 }
 
 int	ft_printf_print_signed(t_printf *ctx, t_format *fmt, int number)
diff --git a/src/ft_numeric_layout.c b/src/ft_numeric_layout.c
new file mode 100644
index 0000000..86657a4
--- /dev/null
+++ b/src/ft_numeric_layout.c
@@ -0,0 +1,50 @@
+#include "ft_printf_internal.h"
+
+static int	ft_prefix_length(const char *prefix)
+{
+	int	length;
+
+	length = 0;
+	while (prefix[length])
+		length++;
+	return (length);
+}
+
+int	ft_printf_write_numeric_layout(t_printf *ctx, t_format *fmt,
+		const char *prefix, const char *digits, int digit_len, int is_zero)
+{
+	int		prefix_len;
+	int		zero_len;
+	int		padding;
+	char	pad_char;
+
+	if (fmt->has_precision && fmt->precision == 0 && is_zero)
+		digit_len = 0;
+	prefix_len = ft_prefix_length(prefix);
+	zero_len = 0;
+	if (fmt->has_precision && fmt->precision > digit_len)
+		zero_len = fmt->precision - digit_len;
+	padding = fmt->width - prefix_len - zero_len - digit_len;
+	pad_char = ' ';
+	if ((fmt->flags & FT_FLAG_ZERO) && !(fmt->flags & FT_FLAG_LEFT)
+		&& !fmt->has_precision)
+		pad_char = '0';
+	if (!(fmt->flags & FT_FLAG_LEFT)
+		&& pad_char == ' '
+		&& ft_printf_putnchar(ctx, ' ', padding) < 0)
+		return (-1);
+	if (ft_printf_write(ctx, prefix, (size_t)prefix_len) < 0)
+		return (-1);
+	if (!(fmt->flags & FT_FLAG_LEFT)
+		&& pad_char == '0'
+		&& ft_printf_putnchar(ctx, '0', padding) < 0)
+		return (-1);
+	if (ft_printf_putnchar(ctx, '0', zero_len) < 0)
+		return (-1);
+	if (ft_printf_write(ctx, digits, (size_t)digit_len) < 0)
+		return (-1);
+	if ((fmt->flags & FT_FLAG_LEFT)
+		&& ft_printf_putnchar(ctx, ' ', padding) < 0)
+		return (-1);
+	return (0);
+}
diff --git a/src/ft_printf_internal.h b/src/ft_printf_internal.h
index 9192c92..5bb0aee 100644
--- a/src/ft_printf_internal.h
+++ b/src/ft_printf_internal.h
@@ -32,6 +32,9 @@ void		ft_printf_init(t_printf *ctx, int fd);
 int			ft_printf_write(t_printf *ctx, const char *buffer, size_t length);
 int			ft_printf_putchar(t_printf *ctx, char c);
 int			ft_printf_putnchar(t_printf *ctx, char c, int length);
+int			ft_printf_write_numeric_layout(t_printf *ctx, t_format *fmt,
+				const char *prefix, const char *digits, int digit_len,
+				int is_zero);
 void		ft_printf_init_format(t_format *fmt);
 const char	*ft_printf_parse(const char *format, t_format *fmt);
 int			ft_printf_dispatch(t_printf *ctx, t_format *fmt, va_list *args);


